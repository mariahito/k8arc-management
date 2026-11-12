# Private-repository change-to-deployment via GitOps (Flux) with GitHub App authentication

This runbook deploys an application to an **already Arc-connected cluster**
from a **private** GitHub repository, authenticated via a GitHub App (not a
PAT, SSH deploy key, or manually generated installation token — Flux uses
the App key to obtain short-lived installation tokens). Arc onboarding and
writing the application/CI pipeline itself are separate prerequisites.

All commands below use Bash, with Azure CLI, GitHub CLI (`gh`), Git, Python
3, `jq`, and `kubectl` installed. Run Azure CLI commands on a machine signed
in to Azure, and `kubectl`/`git`/`gh` commands on a machine with an
authorized kubeconfig and network access to the Kubernetes API and to
GitHub. If Azure and Kubernetes administration happen on separate machines,
run the `az` commands on the Azure administration machine and the `kubectl`
commands on the Kubernetes administration machine; define the relevant
variables on both.

## 1. Set values and verify both targets

```bash
az login
gh auth login --hostname github.com
az extension add --name connectedk8s --upgrade

export SUBSCRIPTION_ID="<your-Azure-subscription-id>"
export RESOURCE_GROUP="<resource-group-containing-your-Arc-cluster>"
export CLUSTER_NAME="<your-Arc-cluster-name>"
export LOCATION="<your-cluster-region>"   # e.g. eastus
export GITHUB_ORG="<your-GitHub-organization>"
export GITHUB_REPO="<your-private-repository-name>"
export REPOSITORY="$GITHUB_ORG/$GITHUB_REPO"
export REPO_URL="https://github.com/$REPOSITORY.git"
export CONFIG_NAME="cicd-demo"
export CONFIG_NAMESPACE="demo-config"
export SECRET_NAME="cicd-demo-github-app"
export APP_NAMESPACE="gitops-demo"
export KUSTOMIZATION_NAME="$CONFIG_NAME-demo"

az account set --subscription "$SUBSCRIPTION_ID"
az account show --query "{name:name,id:id}" -o table
az connectedk8s show -g "$RESOURCE_GROUP" -n "$CLUSTER_NAME" \
  --query "{name:name,connectivity:connectivityStatus,state:provisioningState,location:location}" -o table
gh api user --jq .login
kubectl config current-context
kubectl get nodes
```

Skip sign-in commands if already authenticated. Stop if the account,
cluster, or Kubernetes context is wrong or any command fails. You need Azure
permissions to manage extensions/configurations, Kubernetes permissions to
create the Secret and deploy these workloads, and GitHub permissions to
manage/install the App. The cluster needs outbound access to Azure, the
GitHub API/Git, and your container registry. An App key never belongs in
Git, a transcript, or a chat message.

## 2. Prepare Azure providers and check for an existing Flux extension

The first `az k8s-configuration flux create` automatically installs
`microsoft.flux` if it's absent — the Flux controllers are not a
requirement specific to GitHub Apps.

Check registrations first; run the registration command only for providers
that aren't already `Registered`. Subscription permissions are required.

```bash
az provider show -n Microsoft.Kubernetes --query registrationState -o tsv
az provider show -n Microsoft.KubernetesConfiguration --query registrationState -o tsv
az provider register -n Microsoft.Kubernetes --wait
az provider register -n Microsoft.KubernetesConfiguration --wait

az extension add --name k8s-extension --upgrade
az extension add --name k8s-configuration --upgrade
az k8s-extension list -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" -t connectedClusters -o table
az k8s-configuration flux list -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" -t connectedClusters -o table
```

**If no `microsoft.flux` extension exists**, install it explicitly so the
GitHub provider compatibility check below can run before Step 6:

```bash
az k8s-extension create -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" -t connectedClusters \
  --name flux --extension-type microsoft.flux --scope cluster
```

If Flux already exists, skip that create command. Confirm readiness and
GitHub provider support before continuing — these `kubectl` commands
**inspect/wait for existing controllers; they do not install controllers**:

```bash
az k8s-extension list -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" -t connectedClusters \
  --query "[?extensionType=='microsoft.flux'].{name:name,state:provisioningState,version:version}" -o table
kubectl -n flux-system get deployments
kubectl -n flux-system rollout status deployment/source-controller --timeout=180s
kubectl -n flux-system rollout status deployment/kustomize-controller --timeout=180s
kubectl get crd gitrepositories.source.toolkit.fluxcd.io \
  -o 'jsonpath={.spec.versions[?(@.name=="v1")].schema.openAPIV3Schema.properties.spec.properties.provider.enum}'
```

Expected: extension `Succeeded`, controllers available, and the provider
enum includes `github`. If absent, stop and resolve the managed-extension
version before continuing — do not edit controller Deployments or CRDs as a
workaround.

## 3. Prepare the private repository and CI

This procedure expects your repository to already contain the application
source, a GitOps manifest path (e.g. `./gitops/demo`), and a CI workflow
that publishes generated manifests back to the branch Flux watches.

```bash
gh repo view "$REPOSITORY" --json nameWithOwner,isPrivate,defaultBranchRef
gh repo clone "$REPOSITORY"
cd "$GITHUB_REPO"
git switch main
python3 -m unittest discover -s tests -v
python3 scripts/render.py --check
gh api "repos/$REPOSITORY/actions/permissions"
gh api "repos/$REPOSITORY/actions/permissions/workflow"
gh run list --repo "$REPOSITORY" --workflow gitops.yml --limit 5
```

If you already cloned it, `cd` into that clone instead. Expected: private
repository, `main` branch, passing tests, and current generated manifests.
The CI workflow should grant `contents: write` only to its publishing job
and commit generated manifests — it does not need Azure credentials or the
App key. Repository rules must permit this workflow to update `main`. If
policy blocks it, agree an approved PR-based workflow instead of bypassing
branch protection.

## 4. Register and install the GitHub App (browser steps)

Open the organization-owned registration form:

```
https://github.com/organizations/$GITHUB_ORG/settings/apps/new
```

In the browser:

1. Choose a unique App name, e.g. `<org>-arc-flux`, and enter the repository
   URL as Homepage URL. Set a description identifying its read-only GitOps
   purpose.
2. Leave user authorization/OAuth request and device flow disabled. Uncheck
   Webhook **Active**. No callback URL or webhook receiver is required.
3. Under Repository permissions set **Contents: Read-only**. Metadata
   read-only is mandatory. Do not grant organization/account permissions.
4. Choose **Only on this account** and create the App.
5. In Credentials / Private keys, generate a private key. Store the
   downloaded PEM in a restricted location outside the repository and
   outside any synced cloud folder.
6. Select Install App, choose the organization, choose **Only select
   repositories**, select this repository only, review permissions, then
   install.
7. Record the App ID from the App settings page and the installation ID
   from the final URL `.../settings/installations/<installation-id>`.

Back in your terminal, enter your newly created IDs:

```bash
export APP_ID="<your-App-ID>"
export INSTALLATION_ID="<your-installation-ID>"
export PRIVATE_KEY_PATH="/secure/flux-app.pem"
ls -l "$PRIVATE_KEY_PATH"
```

## 5. Create the configuration namespace and the App Secret

Two supported options to store the GitHub App key as a Kubernetes Secret:
a plain Secret (below — simpler, but not recommended for production), or
Azure Key Vault + Workload Identity Federation (**Step 5b**, recommended).
Pick one before continuing to Step 6 — both produce a Secret with the same
three keys that Step 6 references.

```bash
kubectl get namespaces
# only if $CONFIG_NAMESPACE is absent:
kubectl create namespace "$CONFIG_NAMESPACE"

kubectl -n "$CONFIG_NAMESPACE" create secret generic "$SECRET_NAME" \
  --from-literal="githubAppID=$APP_ID" \
  --from-literal="githubAppInstallationID=$INSTALLATION_ID" \
  --from-file="githubAppPrivateKey=$PRIVATE_KEY_PATH"
kubectl -n "$CONFIG_NAMESPACE" get secret "$SECRET_NAME"
```

Expected: Secret created. `get secret` shows metadata, not key values. If
the Secret already exists, inspect its ownership and use your approved key
rotation procedure instead of overwriting it blindly.

> **This stores the App private key as a plain Kubernetes Secret** — base64
> in etcd, not encrypted at rest unless etcd encryption is separately
> configured, readable by anyone with `get`/`list` RBAC on Secrets in that
> namespace, with no native rotation or audit trail. Use Step 5b instead for
> production.

## 5b. Recommended: Azure Key Vault + Secret Store extension (Workload Identity Federation)

The current, non-retiring, best-practice alternative is Azure's **Secret
Store extension (SSE)** using Workload Identity Federation — not the older
"Azure Key Vault Secrets Provider" extension, which
[retires October 9, 2027](https://learn.microsoft.com/azure/azure-arc/kubernetes/secret-store-extension-migration)
and relies on a static service-principal client secret (itself another
long-lived credential to manage). SSE syncs an Azure Key Vault secret into a
real Kubernetes Secret automatically, authenticating via a federated
identity — no static credential anywhere in the cluster or in Azure AD.

You still need the `$CONFIG_NAMESPACE` namespace to exist first (same
requirement as Step 5):

```bash
kubectl get namespaces   # confirm $CONFIG_NAMESPACE is present; if absent:
kubectl create namespace "$CONFIG_NAMESPACE"
```

With that done, the full SSE sequence, in order:

### 5b.1. Enable the OIDC issuer on the Arc connection

```bash
az connectedk8s update --name "$CLUSTER_NAME" --resource-group "$RESOURCE_GROUP" \
  --subscription "$SUBSCRIPTION_ID" --enable-oidc-issuer --only-show-errors -o json

export ISSUER_URL=$(az connectedk8s show --name "$CLUSTER_NAME" --resource-group "$RESOURCE_GROUP" \
  --query oidcIssuerProfile.issuerUrl -o tsv)
echo "$ISSUER_URL"
```

> If this returns `provisioningState: Failed` with a transient
> service-unavailable/health-check error, simply re-run the same command —
> it typically succeeds on retry.

Verify the issuer's discovery document is actually reachable before
continuing — a newly enabled issuer can take a short time to publish:

```bash
curl -sf "${ISSUER_URL}.well-known/openid-configuration" | head -c 200
```

If this returns a 404 ("container does not exist"), the issuer isn't fully
published yet; re-running `az connectedk8s update --enable-oidc-issuer`
again (triggers a full agent redeploy) typically resolves it within about a
minute.

### 5b.2. Configure `kube-apiserver` with the new issuer

**Applies only to self-managed Kubernetes clusters where you control the
`kube-apiserver` static pod manifest** (not applicable to AKS or other
managed control planes, which expose OIDC issuer configuration through
their own settings). **This step runs on a control-plane node itself**
(SSH there directly — it edits a local file, so your usual
administration/bastion machine won't have it unless it *is* a control-plane
node). Everything below needs root — prefix with `sudo`, and back up the
manifest first; kubelet auto-restarts the apiserver pod when the manifest
changes:

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
sudo sed -i 's#--enable-admission-plugins=NodeRestriction#--enable-admission-plugins=NodeRestriction,OwnerReferencesPermissionEnforcement#' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i "s#--service-account-issuer=.*#--service-account-issuer=${ISSUER_URL}#" /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i '/--service-account-issuer=/a\    - --service-account-max-token-expiration=24h' /etc/kubernetes/manifests/kube-apiserver.yaml
```

> **Important — restarting `kube-apiserver` can break `kube-proxy`'s
> routing to the API server's own ClusterIP.** After any apiserver restart,
> confirm `kube-proxy` is still routing correctly (use your cluster's
> actual `kubernetes` Service ClusterIP, typically `10.96.0.1`):
> ```bash
> curl -sk https://10.96.0.1:443/version
> ```
> If this hangs or refuses the connection while the apiserver pod itself is
> healthy, force `kube-proxy` to resync (run from the control-plane node,
> using its local admin kubeconfig):
> ```bash
> sudo kubectl --kubeconfig=/etc/kubernetes/admin.conf rollout restart daemonset/kube-proxy -n kube-system
> ```
> This is a general risk for any apiserver restart on a self-managed
> cluster — check it after every such change, not just this one.

### 5b.3. Create the Key Vault

```bash
export KV_NAME="<your-Key-Vault-name>"
az keyvault create -n "$KV_NAME" -g "$RESOURCE_GROUP" -l "$LOCATION" --enable-rbac-authorization true
```

> **If your tenant/subscription policy disallows public network access to
> Key Vault**, `--public-network-access Enabled` will appear to succeed and
> then silently revert to `Disabled` on read-back (confirm with
> `az keyvault show --query properties.publicNetworkAccess`). In that case,
> provision a private endpoint instead:
> ```bash
> export VNET_NAME="<your-cluster-VNet-name>"
> export SUBNET_NAME="<your-subnet-name>"
> KV_ID=$(az keyvault show -n "$KV_NAME" -g "$RESOURCE_GROUP" --query id -o tsv)
> az network private-endpoint create -g "$RESOURCE_GROUP" -n "${KV_NAME}-pe" \
>   --vnet-name "$VNET_NAME" --subnet "$SUBNET_NAME" \
>   --private-connection-resource-id "$KV_ID" --group-id vault --connection-name "${KV_NAME}-connection"
> az network private-dns zone create -g "$RESOURCE_GROUP" -n privatelink.vaultcore.azure.net
> az network private-dns link vnet create -g "$RESOURCE_GROUP" -n "${KV_NAME}-dns-link" \
>   -z privatelink.vaultcore.azure.net -v "$VNET_NAME" -e false
> NIC_ID=$(az network private-endpoint show -g "$RESOURCE_GROUP" -n "${KV_NAME}-pe" --query "networkInterfaces[0].id" -o tsv)
> PRIVATE_IP=$(az network nic show --ids "$NIC_ID" --query "ipConfigurations[0].privateIPAddress" -o tsv)
> az network private-dns record-set a create -g "$RESOURCE_GROUP" -z privatelink.vaultcore.azure.net -n "$KV_NAME"
> az network private-dns record-set a add-record -g "$RESOURCE_GROUP" -z privatelink.vaultcore.azure.net -n "$KV_NAME" -a "$PRIVATE_IP"
> ```
> **Consequence: with a private endpoint, all Key Vault data-plane
> operations (`az keyvault secret set`, etc.) only work from a host on the
> cluster's VNet** — run them from a VM/bastion inside that VNet, not
> necessarily your workstation. Management-plane calls (`keyvault create`,
> role assignments, private endpoint/DNS setup) are ARM-level and work from
> anywhere. Confirm DNS resolution before trusting a data-plane call:
> `getent hosts ${KV_NAME}.vault.azure.net` from inside the VNet should
> return the private IP, not fail or resolve publicly.

### 5b.4. Store the App ID, Installation ID, and PEM in Key Vault

Grant yourself (or whoever manages the real GitHub App PEM) Key Vault
Secrets Officer, then store all three as **three separate Key Vault
secrets** (run the `secret set` calls from wherever has Key Vault
data-plane access per the note above). Only the PEM is actually
sensitive — App ID and Installation ID are visible in GitHub's own App
settings UI — but all three go into Key Vault anyway, because combining a
Key-Vault-sourced value with a plain literal value in one SSE-managed
Secret is not supported (see Step 5b.8's note):

```
az ad signed-in-user show --query userPrincipalName -o tsv
UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
```

```bash
KV_ID=$(az keyvault show -n "$KV_NAME" -g "$RESOURCE_GROUP" --query id -o tsv)
az role assignment create --role "Key Vault Secrets Officer" --assignee <your-UPN-or-object-id> --scope "$KV_ID"

az keyvault secret set --vault-name "$KV_NAME" --name github-app-id --value "$APP_ID"
az keyvault secret set --vault-name "$KV_NAME" --name github-app-installation-id --value "$INSTALLATION_ID"
az keyvault secret set --vault-name "$KV_NAME" --name github-app-private-key --file "$PRIVATE_KEY_PATH"
```

### 5b.5. Create a user-assigned managed identity with Key Vault access

```bash
export IDENTITY_NAME="<your-managed-identity-name>"
az identity create --name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" --location "$LOCATION"
IDENTITY_CLIENT_ID=$(az identity show -g "$RESOURCE_GROUP" --name "$IDENTITY_NAME" --query clientId -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show -g "$RESOURCE_GROUP" --name "$IDENTITY_NAME" --query principalId -o tsv)

az role assignment create --role "Key Vault Reader" --assignee-object-id "$IDENTITY_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --scope "$KV_ID"
az role assignment create --role "Key Vault Secrets User" --assignee-object-id "$IDENTITY_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --scope "$KV_ID"
```

> A newly created identity can take ~30-60 seconds to replicate in Entra ID
> before role assignments succeed — "Cannot find user or service principal
> in graph database" if too soon; just wait and retry.

### 5b.6. Federate a Kubernetes service account to the identity

```bash
kubectl create sa cicd-demo-github-app-sa -n "$CONFIG_NAMESPACE"

az identity federated-credential create --name cicd-demo-github-app-fed \
  --identity-name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" \
  --issuer "$ISSUER_URL" \
  --subject "system:serviceaccount:${CONFIG_NAMESPACE}:cicd-demo-github-app-sa" \
  --audience api://AzureADTokenExchange
```

### 5b.7. Install the required extensions (cert-manager must go first)

> **If you changed `kube-apiserver`'s `--service-account-issuer` in Step
> 5b.2 on a cluster that had already been running for a while**, any pod
> still running on a ServiceAccount token minted *before* that change will
> start failing with `Unauthorized` — the token's issuer claim no longer
> matches what the apiserver now expects. This commonly hits the Arc agent
> pods themselves, breaking extension installs with an error like
> `Unable to contact the api-server ... InnerError [Unauthorized]`. Fix:
> restart them so they remount fresh tokens, then retry:
> ```bash
> for d in $(kubectl -n azure-arc get deployment -o jsonpath='{.items[*].metadata.name}'); do
>   kubectl -n azure-arc rollout restart deployment/$d
> done
> kubectl -n azure-arc get pods -w   # wait for all Running with a fresh AGE, then Ctrl+C
> ```
> (`kubectl` itself still works throughout this — the admin kubeconfig
> authenticates via a client certificate, not a ServiceAccount token, so
> it's unaffected.)

```bash
az k8s-extension create --resource-group "$RESOURCE_GROUP" --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters --name azure-cert-management --extension-type microsoft.certmanagement
az k8s-extension create --cluster-name "$CLUSTER_NAME" --cluster-type connectedClusters \
  --extension-type microsoft.azure.secretstore --resource-group "$RESOURCE_GROUP" \
  --name ssarcextension --scope cluster
```

### 5b.8. Sync all three values into one combined Secret

The simplified `AKVSync` CRD only cleanly syncs one object into a
single-key Secret. To combine **three** Key Vault secrets into one Secret
with three specific key names (`githubAppID`, `githubAppInstallationID`,
`githubAppPrivateKey` — exactly what Flux's GitHub App auth expects,
matching Step 5's plain-Secret shape), use the direct
`SecretProviderClass` + `SecretSync` configuration instead.

Get the two placeholder values:

```bash
TENANT_ID=$(az account show --query tenantId -o tsv)
```

Save as a local YAML file, e.g. `cicd-demo-github-app-sync.yaml` — it's a
two-document manifest, same `---`-separated format as any multi-resource
`kubectl apply -f`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: cicd-demo-github-app-spc
  namespace: demo-config
spec:
  provider: azure
  parameters:
    clientID: "<identity-client-id>"
    keyvaultName: <kv-name>
    objects: |
      array:
        - |
          objectName: github-app-id
          objectType: secret
        - |
          objectName: github-app-installation-id
          objectType: secret
        - |
          objectName: github-app-private-key
          objectType: secret
    tenantID: "<tenant-id>"
---
apiVersion: secret-sync.x-k8s.io/v1alpha1
kind: SecretSync
metadata:
  name: cicd-demo-github-app
  namespace: demo-config
spec:
  serviceAccountName: cicd-demo-github-app-sa
  secretProviderClassName: cicd-demo-github-app-spc
  secretObject:
    type: Opaque
    data:
      - sourcePath: github-app-id
        targetKey: githubAppID
      - sourcePath: github-app-installation-id
        targetKey: githubAppInstallationID
      - sourcePath: github-app-private-key
        targetKey: githubAppPrivateKey
```

Substitute `$IDENTITY_CLIENT_ID` / `$KV_NAME` / `$TENANT_ID` into the
placeholders in the file, then apply it:

```bash
envsubst < cicd-demo-github-app-sync.yaml | kubectl apply -f -
```

(Or substitute the placeholders manually with `sed`/an editor, then
`kubectl apply -f cicd-demo-github-app-sync.yaml`.)

This produces a Secret named `cicd-demo-github-app` (same name as the
`SecretSync`) in `demo-config` with exactly the three keys Step 6's
`--local-auth-ref` expects — a drop-in replacement for Step 5's
plain-Secret output, so Step 6 needs no changes.

Verify:

```bash
kubectl get secretsync cicd-demo-github-app -n demo-config -o jsonpath='{.status.conditions[0].message}'
# -> "The secret was updated successfully: a value change or a force update was detected."
kubectl get secret cicd-demo-github-app -n demo-config -o jsonpath='{.data}'
# -> {"githubAppID":"<base64>","githubAppInstallationID":"<base64>","githubAppPrivateKey":"<base64>"}
```

Decode each value and confirm it's an exact byte-for-byte match against
what's stored in Key Vault.

> **Why not just protect the PEM and leave App ID/Installation ID as plain
> Kubernetes values?** There's no supported way to mix a Key-Vault-sourced
> key with a literal/plain key inside one `SecretSync`-managed Secret —
> `SecretSync` owns and reconciles the entire Secret object, so a plain key
> added manually via `kubectl patch` would likely be reverted on the next
> sync. Putting all three in Key Vault (even though only the PEM is
> genuinely sensitive) is the supported path to one combined Secret.

> If the first sync attempt fails with `AADSTS50166: Request to External
> OIDC endpoint failed`, the cluster's OIDC discovery document likely isn't
> published yet — verify it resolves (see Step 5b.1) before troubleshooting
> further.

**Cleanup (if reverting to the plain-Secret approach, or tearing down a
test):**

```bash
az k8s-extension delete --cluster-name "$CLUSTER_NAME" --cluster-type connectedClusters --resource-group "$RESOURCE_GROUP" --name ssarcextension --yes
az k8s-extension delete --cluster-name "$CLUSTER_NAME" --cluster-type connectedClusters --resource-group "$RESOURCE_GROUP" --name azure-cert-management --yes
az identity federated-credential delete --name cicd-demo-github-app-fed --identity-name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP" --yes
az identity delete --name "$IDENTITY_NAME" --resource-group "$RESOURCE_GROUP"
az network private-endpoint delete -g "$RESOURCE_GROUP" -n "${KV_NAME}-pe"
az network private-dns zone delete -g "$RESOURCE_GROUP" -n privatelink.vaultcore.azure.net --yes
az keyvault delete -n "$KV_NAME" -g "$RESOURCE_GROUP"
```

The OIDC issuer itself (`az connectedk8s update --enable-oidc-issuer`) has
no simple disable flag documented; leaving it enabled is harmless if unused.

## 6. Create the Azure configuration and enable its GitHub provider

**New configuration only.** Check the list from Step 2 before creating
anything. Cluster scope grants broad reconciliation permissions for this
demo. `prune=true` permits deletion of resources removed from Git.

```bash
az k8s-configuration flux create \
  --subscription "$SUBSCRIPTION_ID" -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" \
  -t connectedClusters -n "$CONFIG_NAME" --namespace "$CONFIG_NAMESPACE" \
  --scope cluster --url "$REPO_URL" --branch main \
  --local-auth-ref "$SECRET_NAME" --sync-interval 30s \
  --kustomization name=demo path=./gitops/demo prune=true sync_interval=30s timeout=5m
```

Some installed CLI versions only accept `generic`/`azure` for `--provider`,
even though the Azure REST API supports `GitHub` directly. If so, apply
this required follow-up update (this updates Azure's source of truth, not a
controller-managed GitRepository patch). Before it completes, the generic
source may report authentication errors; that intermediate state is not a
successful deployment.

```bash
CLUSTER_ID=$(az connectedk8s show --subscription "$SUBSCRIPTION_ID" \
  -g "$RESOURCE_GROUP" -n "$CLUSTER_NAME" --query id -o tsv)
CONFIG_ID="$CLUSTER_ID/providers/Microsoft.KubernetesConfiguration/fluxConfigurations/$CONFIG_NAME"
CONFIG_URI="https://management.azure.com${CONFIG_ID}?api-version=2025-04-01"

cat > /tmp/flux-github-provider.json <<EOF
{
  "properties": {
    "gitRepository": {
      "url": "$REPO_URL",
      "provider": "GitHub",
      "localAuthRef": "$SECRET_NAME"
    }
  }
}
EOF

az rest --method patch --url "$CONFIG_URI" --body "@/tmp/flux-github-provider.json" \
  --query '{name:name,provider:properties.gitRepository.provider,secret:properties.gitRepository.localAuthRef}' -o json
rm -f /tmp/flux-github-provider.json
```

Expected response: your configuration name, provider `github`, and your
Secret name. The temporary JSON contains only configuration references, not
the App key. If your installed CLI already supports `--provider GitHub`
directly, this REST step may be unnecessary — check before running both.

## 7. Verify private fetch, reconciliation, and application rollout

Creation success alone is not deployment success. First inspect names:

```bash
kubectl -n "$CONFIG_NAMESPACE" get gitrepositories,kustomizations
```

For this procedure the expected names are `$CONFIG_NAME` and
`$KUSTOMIZATION_NAME`. Allow Azure time to create these objects; if their
names differ, use the displayed names rather than guessing.

```bash
kubectl -n "$CONFIG_NAMESPACE" wait "gitrepository/$CONFIG_NAME" \
  --for=condition=ready --timeout=300s
kubectl -n "$CONFIG_NAMESPACE" wait "kustomization/$KUSTOMIZATION_NAME" \
  --for=condition=ready --timeout=300s
kubectl -n "$CONFIG_NAMESPACE" get gitrepository "$CONFIG_NAME" \
  -o 'jsonpath={.spec.provider}{"\n"}{.spec.secretRef.name}{"\n"}{.status.artifact.revision}{"\n"}'
kubectl -n "$APP_NAMESPACE" rollout status deployment/demo-web --timeout=300s
kubectl -n "$APP_NAMESPACE" get deployment demo-web
az k8s-configuration flux show --subscription "$SUBSCRIPTION_ID" \
  -g "$RESOURCE_GROUP" -c "$CLUSTER_NAME" -t connectedClusters -n "$CONFIG_NAME" \
  --query '{compliance:complianceState,error:errorMessage}' -o json
```

Expected: source and Kustomization Ready, provider `github`, correct
Secret, Git SHA, deployment 2/2, and Azure Compliant (Azure status can lag
reconciliation). On failure, stop and inspect conditions without displaying
secrets:

```bash
kubectl -n "$CONFIG_NAMESPACE" describe gitrepository "$CONFIG_NAME"
kubectl -n "$CONFIG_NAMESPACE" describe kustomization "$KUSTOMIZATION_NAME"
kubectl -n "$APP_NAMESPACE" get pods
```

## 8. Read the actual service response

In a **second terminal with the same Kubernetes context**, run:

```bash
kubectl -n gitops-demo port-forward service/demo-web 8080:8080
```

Leave it running. In your original terminal:

```bash
curl -s http://localhost:8080/ | jq
curl -s http://localhost:8080/healthz | jq
```

Expected: release/message match `app/settings.json`; health status is
`ok`. The service remains internal — port-forward does not create public
ingress. Restart port-forwarding after a rollout if its selected pod was
replaced.

## 9. Demonstrate a source change through CI to live deployment (optional)

In your local repository clone, with a clean worktree and an approved
direct-to-main demo policy, push a unique new release. Do not edit the
generated manifests or run the renderer before this push.

```bash
git status --short
git pull --ff-only
export RELEASE="manual-demo-v3"
jq --arg r "$RELEASE" --arg m "Hello from GitOps $RELEASE" '.release = $r | .message = $m' \
  app/settings.json > /tmp/settings.json && mv /tmp/settings.json app/settings.json
git add app/settings.json
git commit -m "Demonstrate source rollout $RELEASE"
SOURCE_COMMIT=$(git rev-parse HEAD)
git push
gh run list --repo "$REPOSITORY" --commit "$SOURCE_COMMIT" --limit 1
```

Copy the run ID for that source commit; if not listed yet, repeat the list
command after a few seconds:

```bash
export RUN_ID="<run-ID-from-the-list>"
gh run watch "$RUN_ID" --repo "$REPOSITORY" --exit-status --interval 10
git pull --ff-only
python3 scripts/render.py --check
EXPECTED_REVISION="main@sha1:$(git rev-parse HEAD)"
```

Wait for the **new bot commit**, not a previous Ready status:

```bash
DEADLINE=$(( $(date +%s) + 300 ))
while [ "$(date +%s)" -lt "$DEADLINE" ]; do
  ACTUAL_REVISION=$(kubectl -n "$CONFIG_NAMESPACE" get kustomization "$KUSTOMIZATION_NAME" \
    -o 'jsonpath={.status.lastAppliedRevision}')
  [ "$ACTUAL_REVISION" = "$EXPECTED_REVISION" ] && break
  sleep 5
done
[ "$ACTUAL_REVISION" = "$EXPECTED_REVISION" ] || { echo "Flux did not apply $EXPECTED_REVISION"; exit 1; }
kubectl -n "$APP_NAMESPACE" rollout status deployment/demo-web --timeout=300s
```

Restart Step 8's port-forward if needed, then assert the live release:

```bash
RESPONSE=$(curl -s http://localhost:8080/)
echo "$RESPONSE"
[ "$(echo "$RESPONSE" | jq -r .release)" = "$RELEASE" ] || { echo "Live application is not the expected release"; exit 1; }
```

For a protected branch/PR workflow, do not push directly: render and commit
the generated output with the source in the PR as your repository
requires.

## 10. Demonstrate source rollback and drift correction (optional)

Roll back the source commit saved in Step 9, not the bot's generated
commit:

```bash
git pull --ff-only
git revert --no-edit "$SOURCE_COMMIT"
git push
ROLLBACK_COMMIT=$(git rev-parse HEAD)
gh run list --repo "$REPOSITORY" --commit "$ROLLBACK_COMMIT" --limit 1
```

Copy the rollback run ID from the list (repeat the list command if the run
hasn't appeared yet), then run:

```bash
export ROLLBACK_RUN_ID="<rollback-run-ID-from-the-list>"
gh run watch "$ROLLBACK_RUN_ID" --repo "$REPOSITORY" --exit-status --interval 10
git pull --ff-only
python3 scripts/render.py --check
RELEASE=$(jq -r .release app/settings.json)
EXPECTED_REVISION="main@sha1:$(git rev-parse HEAD)"
DEADLINE=$(( $(date +%s) + 300 ))
while [ "$(date +%s)" -lt "$DEADLINE" ]; do
  ACTUAL_REVISION=$(kubectl -n "$CONFIG_NAMESPACE" get kustomization "$KUSTOMIZATION_NAME" \
    -o 'jsonpath={.status.lastAppliedRevision}')
  [ "$ACTUAL_REVISION" = "$EXPECTED_REVISION" ] && break
  sleep 5
done
[ "$ACTUAL_REVISION" = "$EXPECTED_REVISION" ] || { echo "Flux did not apply rollback $EXPECTED_REVISION"; exit 1; }
kubectl -n "$APP_NAMESPACE" rollout status deployment/demo-web --timeout=300s
```

Restart Step 8's port-forward if its pod was replaced, then check:

```bash
RESPONSE=$(curl -s http://localhost:8080/)
echo "$RESPONSE"
[ "$(echo "$RESPONSE" | jq -r .release)" = "$RELEASE" ] || { echo "Live application did not roll back"; exit 1; }
```

To demonstrate drift, change only the live replica count:

```bash
kubectl -n "$APP_NAMESPACE" scale deployment/demo-web --replicas=1
kubectl -n "$APP_NAMESPACE" get deployment demo-web -o 'jsonpath={.spec.replicas}'
kubectl -n "$APP_NAMESPACE" get deployment demo-web --watch
```

Observe 1 replica and then Flux restoring the Git-defined 2 on its next
reconciliation interval; Ctrl+C stops the watch, not the workload. This
uses the ordinary reconciliation interval, not a new Git commit. End with
Step 7's rollout/status checks.
