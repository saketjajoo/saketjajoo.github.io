# Securely Accessing GCP Resources from a GKE Pod

*October 13, 2025*

---

Applications on Googler Kubernetes Engine (GKE) may often need to interact with Google Cloud Platform (GCP) services such as Cloud Storage, BigQuery, Pub/Sub, etc. One option to grant such access is by creating a Google Cloud Service Accoun (GSA), download its SA key file (json), store it as a Kubernetes Secret, and mount it to the pod. Although this works fine, it introduces several security risks around Key Management, Key Rotation, Secret Exposure, and more which further adds to operational overheads. The JSON key is a long lived credential that, if compromised, can lead to unauthorized access to GCP resources.

A more secure and manageable way to grant GCP access to applications running on GKE is by using [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity). 


## What is Workload Identity?

Workload Identity allows you to associate a Kubernetes Service Account (KSA) with a Google Cloud Service Account (GSA). Instead of relying on static credentials, the pod inherits the permissions of the GSA when making requests to GCP services. This eliminates the need to manage and distribute service account keys, reducing the risk of key exposure and simplifying credential management.

### Magic behind Workload Identity - STS Token Exchange

A pod running on GKE gets authenticated to GCP via the STS (Security Token Service) token exchange mechanism. This process relies on OAuth2 token exchange orchestrated by the GCP STS service.

1. **Requesting the KSA token**: The GKE metadata server asks the Kubernetes API server for a token for a signed JWT token that represents the pod's KSA. This is a short-lived token (1 hour) with `PROJECT_ID.svc.id.goog` as the audience and is automatically rotated by GKE. 

2. **Exchanging the KSA token for a GSA token**: The GKE metadata server then takes this KSA token and makes a call to the GCP STS API. It essentially says __"Here is a valid, signed token proving my identity within a specific GKE cluster. I want to exchange it for an access token for the GCP SA I'm bound to."__

3. **STS Validation**: STS validates this API request and verifies the IAM binding that allows the KSA to impersonate the GSA (usually given via the `roles/iam.workloadIdentityUser` role by a user).

4. **Issuing the GSA token**: Upon a successful validation, STS issues a short-lived access token (1 hour) for the GSA. This token is then used by the application running in the pod to authenticate to GCP services.

This token exchange process is transparent to the application running in the pod. The application simply uses the Google Cloud client libraries, which automatically handle the token retrieval and refresh process.


### Code Example


#### Pre-requisites

1. [Workload Identity must be enabled]((https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)) on the GKE cluster.

2. [Create a GCP Service Account](https://cloud.google.com/iam/docs/service-accounts-create#creating) and [grant it](https://cloud.google.com/sdk/gcloud/reference/projects/add-iam-policy-binding) the necessary IAM roles to access the required GCP resources.

3. Create a Kubernetes Service Account and [configure](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) the pod to use it. The SA should have a CreateToken permission which can be given via:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
    name: token-requester
    namespace: default
rules:
- apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-serviceaccount-creator
  namespace: default
subjects:
- kind: ServiceAccount
  name: <KSA_Service_Account_Name>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: token-requester
  apiGroup: rbac.authorization.k8s.io
```

4. [Bind the KSA to the GSA](https://cloud.google.com/kubernetes-engine/enterprise/knative-serving/docs/securing/workload-identity#binding_service_accounts).


#### Code Snippet

```go
import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"

    authv1 "k8s.io/api/authentication/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type stsTokenRequest struct {
    GrantType          string
    Audience           string
    Scope              string
    RequestedTokenType string
    SubjectToken       string
    SubjectTokenType   string
}

type stsTokenResponse struct {
    AccessToken     string
    IssuedTokenType string
    TokenType       string
    ExpiresIn       int
}

func exchangeKSATokenForGSAToken(ctx context.Context, k8sClient kubernetes.Interface, projectID, location, clusterName, ksaNamespace, ksaName string) (string, error) {
    ksaAudience := fmt.Sprintf("%s.svc.id.goog", projectID)
    tokenRequest := &authv1.TokenRequest{
        Spec: authv1.TokenRequestSpec{
            Audiences:         []string{ksaAudience},
        },
    }

    resp, err := k8sClient.CoreV1().ServiceAccounts(ksaNamespace).CreateToken(ctx, ksaName, tokenRequest, metav1.CreateOptions{})
    if err != nil {
        return "", fmt.Errorf("failed to create KSA token: %v", err)
    }
    ksaToken := resp.Status.Token

    stsAudience := fmt.Sprintf("identitynamespace:%s.svc.id.goog:https://containers.googleapis.com/v1/projects/%s/locations/%s/clusters/%s", projectID, projectID, location, clusterName)
    stsPayload := &stsTokenRequest{
        Audience:           stsAudience,
        GrantType:          "urn:ietf:params:oauth:grant-type:token-exchange",
        RequestedTokenType: "urn:ietf:params:oauth:token-type:access_token",
        Scope:              "https://www.googleapis.com/auth/cloud-platform",
        SubjectToken:       ksaToken,
        SubjectTokenType:   "urn:ietf:params:oauth:token-type:jwt",
    }
    stsReqBody, err := json.Marshal(stsPayload)
    if err != nil {
        return "", fmt.Errorf("failed to marshal STS request payload: %v", err)
    }

    stsURL := "https://sts.googleapis.com/v1/token"
    stsReq, err := http.NewRequestWithContext(ctx, "POST", stsURL, bytes.NewBuffer(stsReqBody))
    if err != nil {
        return "", fmt.Errorf("failed to create STS request: %v", err)
    }
    stsReq.Header.Set("Content-Type", "application/json")
    stsResp, err := http.DefaultClient.Do(stsReq)
    if err != nil {
        return "", fmt.Errorf("failed to call STS API: %v", err)
    }
    defer stsResp.Body.Close()
    if stsResp.StatusCode != http.StatusOK {
        bodyBytes, _ := io.ReadAll(stsResp.Body)
        return "", fmt.Errorf("STS API returned non-200 status: %d, body: %s", stsResp.StatusCode, string(bodyBytes))
    }

    var gsaToken stsTokenResponse
    if err := json.NewDecoder(stsResp.Body).Decode(&gsaToken); err != nil {
        return "", fmt.Errorf("failed to decode STS response: %v", err)
    }
    return gsaToken.AccessToken, nil
}
```