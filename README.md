# Gator CLI test with Helm charts

Example of using Gator CLI with Helm charts.
This is just a basic example and doesn't showcase all the features of Gatekeeper or Gator CLI.

**Contraint Template and Contraint Files:**

```sh
policies/                         # Contraint templates and contraints go here
  constraint-templates/           # Contraint templates go here
    replicalimits-template.yaml   # Sample contraints template for validating min/max replica limits
  contraints/                     # Specific contraints configured using the contraint templates
    replicalimits.yaml            # Sample contraint for validating min/max replica limits
```

**Charts:**

* Good chart (`app-v1`) - has 3 replicas (see `app-v1/values.yaml`)
* Bad chart  (`app-v2`) - has 1 replica (see `app-v2/values.yaml`)

These were created with:

```sh
helm create app-v1
helm create app-v2
```

Then `app-v2/values.yaml` was edited:

```yaml
replicaCount: 1
```

## Render Helm chart as YAML template

```sh
helm template ./app-v1 | less
helm template ./app-v2 | less
```

If you use `helm upgrade --install app ./app-v1 --dry-run -o yaml` you'll not be able to pass that directly to `gator` since it includes additional information, not just the manifests.  you'd need to strip that out and just select the manifests section.

e.g. error:

```sh
helm upgrade --install app ./app-v1 --dry-run=client -o yaml | gator test --filename=policies/
# auditing objects: adding data of GVK "/, Kind=": admission.k8s.gatekeeper.sh: invalid request object: resource  has no version
```

The output contains additional chart metadata, not just the manifests.

```sh
helm upgrade --install app ./app-v1 --dry-run -o yaml
```

Sample output structure:

```yaml
chart:
  files:
  - data: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    name: .helmignore
  lock: null
  metadata:
    apiVersion: v2
    appVersion: 1.16.0
    description: A Helm chart for Kubernetes
    name: app-v1
    type: application
    version: 0.1.0
  schema: null
  templates:
  - data: xxxxxxxxxxxxxxxxxxxx
  ...
hooks:
- events:
  - test
  kind: Pod
  ...
manifest: |
  ---
  # Source: app-v1/templates/serviceaccount.yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: app-app-v1
    ...
```

The section after `manifest: |` is what we need to pass to `gator`.

## Testing Helm manifests prior to applying to the Kubernetes cluster

```sh
helm template ./app-v1 | gator test --filename=policies/
echo $?
# 0


helm template ./app-v2 | gator test --filename=policies/
# apps/v1/Deployment release-name-app-v2: ["replica-limits"] Message: "The provided number of replicas is not allowed for Deployment: release-name-app-v2. Allowed ranges: {\"ranges\": [{\"max_replicas\": 20, \"min_replicas\": 2}]}"
echo $?
# 1
```

## Resources

* [The gator CLI](https://open-policy-agent.github.io/gatekeeper/website/docs/gator/) - The gator CLI is a tool for evaluating Gatekeeper ConstraintTemplates and Constraints in a local environment.
