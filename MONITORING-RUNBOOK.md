# Azure Monitor managed Prometheus + Grafana + Container Insights + Flux monitoring — setup runbook

This runbook integrates Azure Monitor managed Prometheus, Azure Managed
Grafana, container logging (Container Insights), and GitOps/Flux-specific
monitoring on an Arc-enabled Kubernetes cluster.

Supporting files referenced below (included alongside this runbook):

- `scripts/ama-metrics-settings-configmap.yaml`
- `scripts/flux-kube-state-metrics.yaml`
- `scripts/flux-alert-rules.json`
- `reference/flux-dashboards/*.json` (3 dashboard definitions)

Sources: [Enable monitoring for Arc-enabled Kubernetes clusters](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable-arc?tabs=cli),
[Link a Grafana workspace](https://learn.microsoft.com/en-us/azure/managed-grafana/quickstart-managed-grafana-portal),
[Monitor GitOps (Flux v2) status and activity](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/monitor-gitops-flux-2).

All commands below use Bash with the Azure CLI, `kubectl`, and `jq` installed.
Run Azure CLI (`az ...`) commands on a machine signed in to Azure; run
`kubectl` commands on a machine with an authorized kubeconfig for the target
cluster. If those are different machines, run each command on the
appropriate one.

## 1. Set environment variables

```bash
export SUBSCRIPTION_ID="<your-subscription-id>"
export RESOURCE_GROUP="<resource-group-containing-your-Arc-cluster>"
export CLUSTER_NAME="<your-Arc-cluster-name>"
export LOCATION="<your-cluster-region>"   # e.g. eastus

az account set --subscription "$SUBSCRIPTION_ID"
az connectedk8s show -g "$RESOURCE_GROUP" -n "$CLUSTER_NAME" \
  --query "{name:name,connectivityStatus:connectivityStatus}" -o table
```

## 2. Confirm prerequisites

- Contributor access to the cluster (for onboarding the extensions below).
- Monitoring Reader/Contributor (to view data afterward).
- Required resource providers registered in the subscription:
  `Microsoft.ContainerService`, `Microsoft.Insights`,
  `Microsoft.AlertsManagement`, `Microsoft.Monitor`.

```bash
for p in Microsoft.ContainerService Microsoft.Insights Microsoft.AlertsManagement Microsoft.Monitor; do
  echo "$p: $(az provider show -n "$p" --query registrationState -o tsv)"
done
```

Register any provider that isn't already `Registered`:

```bash
az provider register --namespace <provider-name> --wait
```

Install the required CLI extension:

```bash
az extension add --name k8s-extension --upgrade
```

## 3. Create the managed Prometheus extension

This installs the extension and lets Azure auto-create (or reuse) a regional
Azure Monitor workspace:

```bash
az k8s-extension create --name azuremonitor-metrics \
  --cluster-name "$CLUSTER_NAME" --resource-group "$RESOURCE_GROUP" \
  --cluster-type connectedClusters \
  --extension-type Microsoft.AzureMonitor.Containers.Metrics -o json
```

This can take several minutes — it deploys the `ama-metrics-*` pods and
provisions a `Microsoft.Monitor/accounts` resource plus the supporting data
collection rule/endpoint.

By default Azure creates/reuses a workspace named
`DefaultAzureMonitorWorkspace-<region>` in a resource group named
`DefaultResourceGroup-<region>` — this is expected Azure behavior (the
workspace is a regional singleton per subscription+region), not an error. Use
`az monitor account list -o table` to find its exact name/resource group
rather than assuming the naming pattern.

To use an existing Azure Monitor workspace instead, add:

```bash
--configuration-settings azure-monitor-workspace-resource-id=<workspace-resource-id>
```

(optionally plus `grafana-resource-id=<grafana-resource-id>` to link Managed
Grafana at the same time as Step 5 below).

## 4. Verify deployment

```bash
kubectl get ds ama-metrics-node -n kube-system
kubectl get rs -n kube-system | grep ama-metrics
```

Expect `ama-metrics-node` DESIRED/CURRENT/READY to match your node count, and
the `ama-metrics-*` / `ama-metrics-ksm-*` ReplicaSets Ready, all pods
`Running`.

Confirm the extension and the Azure Monitor workspace:

```bash
az k8s-extension show --name azuremonitor-metrics --cluster-name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" --cluster-type connectedClusters \
  --query "{name:name,provisioningState:provisioningState,extensionType:extensionType}" -o table
az monitor account list -o table
```

## 5. Link Azure Managed Grafana

### 5a. Install the CLI extension and confirm the provider

```bash
az extension add --name amg --upgrade --only-show-errors
az provider show -n Microsoft.Dashboard --query registrationState -o tsv
```

### 5b. Create the Grafana workspace

```bash
export GRAFANA_NAME="<your-Grafana-workspace-name>"
az grafana create -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" -o json
```

### 5c. Point the Prometheus extension at the workspace + Grafana resource

```bash
export AMW_RESOURCE_GROUP="<Azure-Monitor-workspace-resource-group>"   # from Step 3/4
export AMW_NAME="<Azure-Monitor-workspace-name>"                       # from Step 3/4
AMW_ID=$(az monitor account show -g "$AMW_RESOURCE_GROUP" -n "$AMW_NAME" --query id -o tsv)
GRAFANA_ID=$(az grafana show -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --query id -o tsv)

az k8s-extension update --name azuremonitor-metrics --cluster-name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" --cluster-type connectedClusters \
  --configuration-settings \
    azure-monitor-workspace-resource-id="$AMW_ID" \
    grafana-resource-id="$GRAFANA_ID" \
  --yes -o json
```

`--yes` is required for non-interactive use — without it the CLI prompts
"Do you wish to proceed? (y/n)" and fails under automation.

### 5d. Grant Grafana's identity read access to the Prometheus data (manual step)

> Linking in 5c does **not** automatically grant Grafana's managed identity
> the `Monitoring Data Reader` role on the Azure Monitor workspace — without
> this, dashboards will fail to load data with an access error. Create it
> explicitly:

```bash
GRAFANA_PRINCIPAL_ID=$(az grafana show -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --query identity.principalId -o tsv)
az role assignment create --assignee-object-id "$GRAFANA_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --role "Monitoring Data Reader" \
  --scope "$AMW_ID"
```

Requires `Owner` (or `User Access Administrator`) on the Azure Monitor
workspace's resource group.

### 5e. Add the Azure Monitor workspace as a Grafana data source

```bash
az grafana integration monitor add -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" \
  --monitor-name "$AMW_NAME" --monitor-rg-name "$AMW_RESOURCE_GROUP"
```

(`az grafana integration monitor` is `[Preview]`.) Note the flag is
`--monitor-rg-name`, not `--monitor-resource-group`.

### 5f. Verify the link

```bash
az grafana integration monitor list -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" -o json
```

### 5g. Open Grafana and use the prebuilt dashboards

```bash
az grafana show -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --query properties.endpoint -o tsv
```

Open the returned URL and sign in with your Azure AD account. In Grafana:
**Dashboards** → expand the **Managed Prometheus** data source → open a
prebuilt dashboard (e.g. "Kubernetes / Compute Resources / Cluster", "Node
Exporter / Nodes", "Overview"). Grafana access is via Azure RBAC
(`Grafana Viewer`/`Grafana Editor`/`Grafana Admin` roles on the Grafana
resource) — separate from the `Monitoring Data Reader` role granted in 5d
(that one lets Grafana itself read the metrics; RBAC on the Grafana resource
controls who can log into Grafana).

## 6. Container logs (Container Insights)

Installs the logs add-on with the default Log Analytics workspace and
managed identity auth:

```bash
az k8s-extension create --name azuremonitor-containers --cluster-name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" --cluster-type connectedClusters \
  --extension-type Microsoft.AzureMonitor.Containers -o json
```

Azure auto-creates/uses a default Log Analytics workspace — it may land in a
different resource group than the Prometheus workspace from Step 3; use
`az monitor log-analytics workspace list -o table` to find it rather than
assuming a naming pattern.

Verify deployment:

```bash
kubectl get ds ama-logs -n kube-system
kubectl get deployment ama-logs-rs -n kube-system
kubectl get pods -n kube-system | grep ama-logs
```

Expect `ama-logs` daemonset ready count to match your node count, and
`ama-logs-rs` deployment `1/1`.

Not covered in this runbook (see the source doc for details): custom
resource limits/mount paths, `dataCollectionSettings` namespace filtering,
forward-proxy configuration, and the Windows-node legacy-auth variant.
Default settings (`interval: 1m`, all namespaces, ContainerLogV2 enabled)
apply.

## 7. Flux-specific monitoring

Source: [Monitor GitOps (Flux v2) status and activity](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/monitor-gitops-flux-2).
Three distinct capabilities, covered in 7a-7e below.

### 7a. Portal GitOps view — no action needed

Because the `microsoft.flux` extension and a Flux configuration already
exist, the Azure portal's cluster **GitOps** blade already shows compliance
state, configuration objects, and the Flux configuration Overview page.

### 7b. Enable Flux controller metrics scraping

Flux's controllers expose Prometheus metrics via pod annotations, but
nothing scrapes them by default. Apply the ConfigMap at
`scripts/ama-metrics-settings-configmap.yaml` (schema v2 — the current
recommended schema, nesting settings under `cluster-metrics:` /
`controlplane-metrics:`). Before applying, edit its
`podannotationnamespaceregex` value to match your own Flux/application
namespaces, e.g. `"flux-system|<your-app-namespace>"`.

```bash
kubectl apply -f scripts/ama-metrics-settings-configmap.yaml
kubectl rollout restart daemonset/ama-metrics-node -n kube-system
```

The ConfigMap alone takes effect within ~2-3 minutes as `ama-metrics-node`
auto-reloads; the rollout restart just applies it immediately.

### 7c. Deploy kube-state-metrics for Flux

Flux removed its `gotk_reconcile_condition`/`gotk_suspend_status`
Prometheus metrics in v2.1, replacing them with `gotk_resource_info`
collected via kube-state-metrics' custom-resource-state feature (see
"Background" at the end of this section). Deploy a dedicated instance
scoped to the Flux CRDs — manifest at `scripts/flux-kube-state-metrics.yaml`
(RBAC, ConfigMap, Deployment, Service; runs in `flux-system` so the Step 7b
scrape config picks it up automatically, no extra scrape config needed):

```bash
kubectl apply -f scripts/flux-kube-state-metrics.yaml
```

Verify:

```bash
kubectl get pods -n flux-system -l app=flux-kube-state-metrics
```

Expect `1/1 Running`. Allow ~2 minutes for the scrape + remote-write to land
in the Azure Monitor workspace before checking dashboards.

### 7d. Import the dashboards

```bash
cd reference/flux-dashboards
az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'GitOps Flux - Application Deployments Dashboard.json' -o json
az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'Flux Control Plane.json' -o json
az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'Flux Cluster Stats.json' -o json
```

This imports three dashboards: `gitops-application-deployments`,
`flux-control-plane`, `flux-cluster`.

**Only remaining per-environment step:** each dashboard's datasource
template variable (`DS_PROMETHEUS` / `DS_FLUX_GITOPS`) must point at your
actual Grafana datasource names — the bundled JSON's defaults won't match a
new Grafana instance, so the dashboards will show "No data" until
re-pointed. Two ways to fix it:

**Option A — in the Grafana UI (quickest):** open each imported dashboard,
use the **Datasource**/**Azure Monitor Data Source** dropdown at the
top-left, and select your actual datasource. Grafana saves that selection
with the dashboard automatically.

**Option B — patch the JSON before importing (for a scripted/repeatable
deployment).** First look up your real datasource names:

```bash
az grafana data-source list -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" -o table
# Note the "Name" column values for the Prometheus datasource and for "Azure Monitor"
```

Then patch the JSON and import:

```bash
cd reference/flux-dashboards
export PROM_NAME="<your-Prometheus-datasource-Name>"
export AZMON_NAME="<your-Azure-Monitor-datasource-Name, usually 'Azure Monitor'>"

for f in "Flux Control Plane.json" "Flux Cluster Stats.json"; do
  jq --arg name "$PROM_NAME" \
    '(.templating.list[] | select(.name=="DS_PROMETHEUS") | .current.text) = $name
     | (.templating.list[] | select(.name=="DS_PROMETHEUS") | .current.value) = $name' \
    "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done

f="GitOps Flux - Application Deployments Dashboard.json"
jq --arg name "$AZMON_NAME" \
  '(.templating.list[] | select(.name=="DS_FLUX_GITOPS") | .current.text) = $name
   | (.templating.list[] | select(.name=="DS_FLUX_GITOPS") | .current.value) = $name' \
  "$f" > "$f.tmp" && mv "$f.tmp" "$f"

az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'GitOps Flux - Application Deployments Dashboard.json' -o json
az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'Flux Control Plane.json' -o json
az grafana dashboard import -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --definition 'Flux Cluster Stats.json' -o json
```

Verify it took effect:

```bash
az grafana dashboard show -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --dashboard flux-control-plane \
  --query "dashboard.templating.list[?name=='DS_PROMETHEUS'].current" -o json
```

**Background: why the bundled JSON needed fixing (reference only, already
applied in the files provided with this runbook)**

The dashboards as published upstream at
[Azure/fluxv2-grafana-dashboards](https://github.com/Azure/fluxv2-grafana-dashboards)
have three problems, already fixed in the files under
`reference/flux-dashboards/`:

1. **Non-null `id` field** in `Flux Control Plane.json` / `Flux Cluster
   Stats.json` makes `az grafana dashboard import` return 404 trying to
   update a dashboard by that literal numeric ID. Fixed by nulling `id`.
2. **Broken datasource defaults** — `DS_PROMETHEUS` defaults to a
   placeholder name that won't exist in your environment, and the `GitOps
   Flux - Application Deployments Dashboard.json` variable is misnamed
   upstream (`GitOpsDataSource` defined, but panels reference
   `${DS_FLUX_GITOPS}`). Fixed by correcting the variable name; you still
   need to re-point the *value* for your own environment (Step 7d above).
3. **Deprecated metric** — `Flux Cluster Stats.json`'s
   "Reconcilers"/"Sources"/readiness panels queried
   `gotk_reconcile_condition`, removed from Flux upstream in v2.1
   ([fluxcd/flux2#5083](https://github.com/fluxcd/flux2/issues/5083)) in
   favor of `gotk_resource_info` via kube-state-metrics
   ([fluxcd.io/flux/monitoring/custom-metrics](https://fluxcd.io/flux/monitoring/custom-metrics/)).
   Fixed by deploying `flux-kube-state-metrics` (Step 7c) and rewriting the
   affected panels/variables to the new metric. Two Azure-ingestion-specific
   adjustments are baked into the fix: Azure Monitor's ingestion pipeline
   lowercases Info-metric label values (`ready="True"` → `ready="true"`,
   `HelmRepository` → `helmrepository`), and auto-injects a `cluster` label
   on every ingested series (not present at the raw pod `/metrics` level).
   `flux-kube-state-metrics` also needs explicit RBAC to `list`/`watch`
   `customresourcedefinitions` (cluster-scoped) — already included in the
   manifest.
4. **"No data" on the two `stat` panels ("Failing Reconcilers", "Failing
   Sources") when nothing is actually failing.** Because there are
   genuinely zero `ready="false"` series when everything is healthy, these
   panels show "No data" — indistinguishable by appearance from a broken
   query. Fixed by adding `or vector(0)` to both queries so they display an
   explicit `0` instead. Deliberately **not** applied to the two `table`
   panels ("Cluster reconciliation readiness", "Source acquisition
   readiness") — those list failing resources by name, and `vector(0)`
   would insert a spurious unlabeled row; "No data" is the correct empty
   state there.

### 7e. Alerts for resource-consumption/reconciliation issues

Source doc section: [Create alerts for resource consumption and
reconciliation issues](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/monitor-gitops-flux-2#create-alerts-for-resource-consumption-and-reconciliation-issues).

> **Note:** the source doc's own sample Bicep template queries
> `gotk_reconcile_condition` — the same deprecated metric from 7d's
> "Background". The rules file provided with this runbook
> (`scripts/flux-alert-rules.json`) already uses `gotk_resource_info`
> instead, so these alerts will actually fire.

These are pure Azure Resource Manager resources
(`Microsoft.Insights/actionGroups`,
`Microsoft.AlertsManagement/prometheusRuleGroups`) reading the Azure Monitor
workspace already populated above — nothing on the cluster is touched by
this step.

**Create the action group** (email notification target):

```bash
az monitor action-group create -n fluxRuleActionGroup -g "$RESOURCE_GROUP" \
  --short-name fluxGroup --action email emailReceiver <your-email> -o json
```

**Update the rules file with your action group**, then create the rule
group. `scripts/flux-alert-rules.json` mirrors 5 alerts from the source
doc (`KustomizationNotReady`, `HelmReleaseNotReady`, `GitRepositoryNotReady`,
`BucketNotReady`, `HelmRepositoryNotReady`), each rewritten to use
`gotk_resource_info` with the corresponding label names
(`exported_namespace` instead of `namespace`, `customresource_kind` instead
of `kind`, lowercase kind values, `ready="false"` instead of
`type="Ready", status="False"`). Each rule's `actions[].actionGroupId` must
point at the action group you just created:

```bash
ACTION_GROUP_ID=$(az monitor action-group show -n fluxRuleActionGroup -g "$RESOURCE_GROUP" --query id -o tsv)
jq --arg id "$ACTION_GROUP_ID" '(.[] | .actions[0].actionGroupId) = $id' \
  scripts/flux-alert-rules.json > /tmp/flux-alert-rules.json

az extension add --name alertsmanagement --upgrade --only-show-errors
az alerts-management prometheus-rule-group create -n fluxRuleGroup -g "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --scopes "$AMW_ID" \
  --rules @/tmp/flux-alert-rules.json --description "Flux Prometheus Rule Group" \
  --cluster-name "$CLUSTER_NAME" -o json
```

`--rules` must reference a `.json` file (not `.yaml` — some CLI versions
fail to auto-detect YAML and try to parse it as shorthand key=value syntax).
`--cluster-name` sets the `cluster` label value the rule group expects on
incoming metrics (must match what's actually ingested — see Step 7c).
`--scopes` must be the Azure Monitor workspace resource ID (not the
cluster), the same workspace used throughout this runbook.

Verify no alerts have fired unexpectedly:

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.AlertsManagement/alerts?api-version=2019-05-05-preview" \
  --query "value[?contains(properties.essentials.alertRule, 'NotReady')]" -o json
```

**Not covered in this runbook:** the source doc's companion "Create alerts
for extension and configuration failures" section — that uses Grafana's own
native alerting UI on the Application Deployments dashboard (not an ARM
resource).

## 8. Cleanup

```bash
# Remove the imported Flux dashboards (reverses Step 7d)
for uid in gitops-application-deployments flux-control-plane flux-cluster; do
  az grafana dashboard delete -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --dashboard "$uid"
done

# Remove the flux-kube-state-metrics deployment (reverses Step 7c)
kubectl delete -f scripts/flux-kube-state-metrics.yaml

# Remove the Flux scrape ConfigMap (reverses Step 7b)
kubectl delete configmap ama-metrics-settings-configmap -n kube-system

# Remove the Flux alert rule group + action group (reverses Step 7e)
az alerts-management prometheus-rule-group delete -n fluxRuleGroup -g "$RESOURCE_GROUP" --yes
az monitor action-group delete -n fluxRuleActionGroup -g "$RESOURCE_GROUP"

# Remove the Grafana integration + role assignment (reverses Step 5d/5e)
az grafana integration monitor delete -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" \
  --monitor-name "$AMW_NAME" --monitor-rg-name "$AMW_RESOURCE_GROUP"
az role assignment delete --assignee-object-id "$GRAFANA_PRINCIPAL_ID" \
  --role "Monitoring Data Reader" --scope "$AMW_ID"

# Remove the Grafana workspace
az grafana delete -g "$RESOURCE_GROUP" -n "$GRAFANA_NAME" --yes

# Remove the Prometheus extension (pods only — does not delete the workspace resource)
az k8s-extension delete --name azuremonitor-metrics --cluster-name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" --cluster-type connectedClusters --yes

# Remove the container-logs extension (workspace data is left intact)
az k8s-extension delete --name azuremonitor-containers --cluster-name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" --cluster-type connectedClusters --yes
```

Neither the Azure Monitor workspace (`Microsoft.Monitor/accounts`) nor the
Log Analytics workspace (`Microsoft.OperationalInsights/workspaces`) is
deleted by the above, and both are separately billable; delete explicitly if
no longer needed:

```bash
az monitor account delete -g "$AMW_RESOURCE_GROUP" -n "$AMW_NAME" --yes
az monitor log-analytics workspace delete -g <log-analytics-resource-group> -n <log-analytics-workspace-name> --yes --force
```
