# Understanding the Kubernetes Reconciler Loop: `Generation` vs `ObservedGeneration`

*October 10, 2025*

---

If you've ever built a Kubernetes operator or delved into the internals of how Kubernetes controllers manage [Custom Resources (CRs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), you've likely encountered the terms `Generation` and `ObservedGeneration`. At first glance, they seem quite similar, but understanding their roles is crucial for writing robust controllers that avoid infinite reconciliation loops.

## When to Reconcile?

A Kubernetes controller's main job is to ensure that the _actual_ state of any object matches the _desired_ state. This _desired_ state is mostly declared by users through the `Spec` field of a resource; but can also be influenced by custom business logic written in the controller's reconciliation itself. The controller continuously `watch`es for changes to the objects managed by it, and acts to bring the actual state in line with the desired state.

Here's the catch - upon taking some action to bring the actual state closer to (ideally equal to) the desired state, the controller may often update the object itself to reflect the change (e.g. updating the `Status` field). This update to the object will trigger another reconciliation loop because the object has changed. If not handled properly, this can lead to an infinite loop of reconciliations.

This is where `Generation` and `ObservedGeneration` come into play.

<br/>

### `metadata.Generation`: the desired state

The `metadata.Generation` field is managed by the Kubernetes API server. It is a monotonically increasing integer, starting from `1`, and incremented every time there is a change to mutable fields in the object's `Spec`.

* **Who sets it**: The Kubernetes API server.
* **When does it change**: Whenever any client (e.g. a user, `kubectl`, any other controller, client libraries, etc.) updates the mutable fields in the `Spec` of the object, other metadata changes (e.g., labels or annotations) do not increment `Generation`.
* **What does it represent**: The _version_ of the _desired_ state as last updated.

<br/>

### `status.ObservedGeneration`: the controller's ACK

The `status.ObservedGeneration` field is managed by the controller itself. When a controller successfully processes a specific `metadata.Generation` of an object, it updates the `status.ObservedGeneration` field to be equal to the `metadata.Generation` it just processed.

> Storing `ObservedGeneration` in the status field ensures that updates to it don’t accidentally change the desired state of the object, and avoids unnecessary `Generation` increments.”

* **Who sets it**: The controller.
* **When does it change**: Whenever the controller successfully reconciles the state of an object corresponding to a particular `metadata.Generation`.
* **What does it represent**: The _version_ of the _desired_ state that the controller has successfully observed and acted upon.

<br/>

## How do they work together in the Reconciler Loop?

The typical pattern of a controller's `Reconcile` loop looks like this:

1. **Receive Event**: The controller receives an event indicating that an object has changed.

2. **Fetch Object**: The controller fetches the latest state of the object from the Kubernetes API server.

3. **Compare Generation**: The controller compares the object's `metadata.Generation` with its `status.ObservedGeneration`.
   
   * If `metadata.Generation == status.ObservedGeneration`: It means that the controller has already processed the current desired state. The controller can **skip reconciliation** for this event, avoiding unnecessary work.
   
   * If `metadata.Generation != status.ObservedGeneration`: It means that the object's desired state has been updated since it was last processed. The controller must **run through its reconciliation loop** to bring the object's actual state in line with the new desired state.

4. **Reconcile State**: The controller performs the necessary actions to reconcile the actual state with the desired state, if needed.

5. **Update ObservedGeneration**: After successfully reconciling the object, the controller updates the `status.ObservedGeneration` to match the `metadata.Generation` it just processed.
   * This update triggers another reconciliation event because the object has technically changed, but since `metadata.Generation` and `status.ObservedGeneration` are now equal, the controller skips further reconciliation for this event.


```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    object := &v1.MyCustomResource{}
    if err := r.Get(ctx, req.NamespacedName, object); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // Check if we need to reconcile
    if object.Generation == object.Status.ObservedGeneration {
        r.Log.Info("Skipping reconciliation as Generation matches ObservedGeneration", "Generation", object.Generation, "ObservedGeneration", object.Status.ObservedGeneration)
        return ctrl.Result{}, nil
    }

    r.Log.Info("Reconciling", "Generation", object.Generation, "ObservedGeneration", object.Status.ObservedGeneration)
    //  Reconcile the object
    object.Status.ObservedGeneration = object.Generation
    if err := r.Status().Update(ctx, object); err != nil {
        r.Log.Error(err, "Failed to update ObservedGeneration")
        return ctrl.Result{}, err
    }
    r.Log.Info("Successfully reconciled and updated ObservedGeneration", "ObservedGeneration", object.Status.ObservedGeneration, "Generation", object.Generation)
    return ctrl.Result{}, nil
}
```

## Conclusion

`metadata.Generation` and `status.ObservedGeneration` are subtle but crucial and powerful tools in the Kubernetes controller pattern. By leveraging these fields, controllers can efficiently determine when to reconcile an object, preventing unnecessary work and avoiding infinite loops.
