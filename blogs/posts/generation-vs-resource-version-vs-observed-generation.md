# Generation vs Resource Version vs Observed Generation in Kubernetes

*October 28, 2025*

---

When you `kubectl apply` to any other Kubernetes resource, a complex series of operations begins between you, the API server, and the various controllers that make the cluster work; the end result - a new object gets created/updated. But how does the system keep track of all these changes? How does a controller know it has work to do, and how do you know the object is ready for consumption?

The answer lies in three metadata fields that are critical: `metadata.generation`, `metadata.resourceVersion`, and `status.observedGeneration`. While they all sound like "versions", they serve distinct purposes.

## resourceVersion: The "When"
The [resourceVersion](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions) is the most granular field of the three.
* **What**: It is a string (not a number) that represents the internal version of the object in etcd.
* **When does it change**: It changes with _every_ modification to an object, whether it's a change in `spec`, `status` or `metadata`.
* **Why**:
  * **Optimistic Concurrency Control**: When an object is updated, the `resourceVersion` of the object is sent. If the `resourceVersion` in etcd is different (meaning someone else changed it since it was last read), the API server will reject this update. This prevents from accidentally overwriting someone else's changes.
  * **The `watch` mechanism**: Kubernetes controllers use `watch` to subscribe to changes. They tell the API server "_send me all changes for an object starting from resourceVersion X_". The API server then streams all subsequent changes (adds, modifications, deletes) for that object, allowing controllers to react without constantly polling the entire cluster.

### resourceVersion = "0": A Special Case  
* `get` / `list`: When the `resourceVersion` is set to `"0"` in a `get` or a `list` request, it tells the API server to return the object at _any_ resource version. This means that the caller is willing to accept a stale object and the API server serves this object from its cache instead of reading it from etcd.

* `watch` (List-Then-Watch): The API server first does a list of _all_ current objects and sends those disguised as `ADD` events via which a controller populates its local cache. After this, the API server sends a `BOOKMARK` event with a recent, real `resourceVersion`. From this point onwards, it streams live `ADD`, `MODIFY`, and `DELETE` events.


## generation: The "What"
The `metadata.generation` is a much more specific and user-facing field. It tracks the intent.
* **What**: It is an integer that represents a specific version of the desired state of the object, as defined by its `spec`.
* **When does it change**: It increments _only_ when a change is made to an object's `spec` field. Any changes to `status` or `metadata` do not affect the `generation`.
* **Why**: It is primarily used by controllers to determine if they need to take action. When a controller sees that the `generation` has increased, it knows that the desired state has changed and it needs to reconcile the actual state to match the new desired state.

## observedGeneration: The "Done"
When a controller processes an object when it sees a change in `generation`, it updates the `status.observedGeneration` field to reflect that it has observed and acted upon that generation.
* **What**: It is an integer that indicates the most recent `generation` that the controller has processed.
* **When does it change**: It is updated by the controller after it has successfully reconciled the object to match the desired state defined by the `spec`.
* **Why**: It provides a way for users and other controllers to know whether the current state of the object reflects the desired state.

## Tying It All Together

<div class="mermaid">
sequenceDiagram
    participant User
    participant APIServer
    participant Controller
    Note over User, Controller: Initial State: generation=0, observedGeneration=0
    User->>APIServer: 1. kubectl apply
    APIServer->>APIServer: 2. Create Deployment
    APIServer-->>User: (ack)
    Note right of APIServer: generation=1, resourceVersion="1001"
    APIServer->>Controller: 3. Watch Event (New Deployment)
    Controller->>Controller: 4. Reconcile: generation (1) > observedGeneration (0)
    Controller->>Controller: 5. Create ReplicaSet, Pods...
    Controller->>APIServer: 6. Update Deployment Status
    Note right of APIServer: status.observedGeneration=1, resourceVersion="1005"
    APIServer-->>Controller: (ack)
    Note over User, Controller: System Stable: generation=1, observedGeneration=1
    User->>APIServer: 7. kubectl apply (spec change)
    APIServer->>APIServer: 8. Update Deployment Spec
    APIServer-->>User: (ack)
    Note right of APIServer: generation=2, resourceVersion="1090"
    APIServer->>Controller: 9. Watch Event (Modified Deployment)
    Controller->>Controller: 10. Reconcile: generation (2) > observedGeneration (1)
    Controller->>Controller: 11. Start Rolling Update...
    Controller->>APIServer: 12. Update Deployment Status
    Note right of APIServer: status.observedGeneration=2, resourceVersion="1150"
    APIServer-->>Controller: (ack)
    Note over User, Controller: System Stable: generation=2, observedGeneration=2
</div>