# Argo CD ApplicationSet + Rancher + Remote Hosting Cluster

This package assumes:
- Rancher and Argo CD run in the management environment.
- Customer workloads run on a DIFFERENT Kubernetes cluster.
- Argo CD manages applications on that remote cluster.

## 1. Register the remote hosting cluster in Argo CD

Make sure your kubeconfig has a context for the remote cluster:

```bash
kubectl config get-contexts
kubectl --context <REMOTE_CONTEXT> get nodes
```

Register it with Argo CD using the name expected by the ApplicationSet:

```bash
argocd cluster add <REMOTE_CONTEXT> --name hosting-cluster
```

Verify:

```bash
argocd cluster list
```

The remote cluster must appear as:

```text
hosting-cluster
```

## 2. Update the Git URL

Edit:

```text
applicationset/customers.yaml
```

Replace both occurrences of:

```text
https://github.com/YOUR-ORG/customer-apps.git
```

with your repository URL.

## 3. Push this repository to Git

Argo CD/ApplicationSet reads the customer files and Helm chart from Git.

## 4. Apply the ApplicationSet on the Argo CD management cluster

```bash
kubectl apply -f applicationset/customers.yaml
```

The ApplicationSet object belongs in the cluster where Argo CD runs.

## 5. Result

Argo CD generates:

```text
cust001-appy-bbbbb
cust002-appy-bbbbb
cust003-appy-bbbbb
```

All target:

```text
hosting-cluster
```

Argo CD automatically creates the namespaces:

```text
cust001
cust002
cust003
```

Each namespace gets:

```text
<customer>-appy-bbbbb-nginx
<customer>-appy-bbbbb-ping
```

## 6. Verify on the remote hosting cluster

```bash
kubectl --context <REMOTE_CONTEXT> get ns
kubectl --context <REMOTE_CONTEXT> get all -n cust001
```

## 7. Test the ping service

```bash
kubectl --context <REMOTE_CONTEXT> run curl-test   -n cust001   --rm -it   --restart=Never   --image=curlimages/curl   -- curl http://cust001-appy-bbbbb-ping
```

Expected response:

```text
cust001-appy-bbbbb-ping
```

## 8. Add a new customer

Create:

```text
customers/cust004.yaml
```

with:

```yaml
customerId: cust004
applicationName: cust004-appy-bbbbb
namespace: cust004
```

Commit and push. ApplicationSet creates the new Application and namespace on
the remote hosting cluster.

## Important

The Argo CD application controller must be able to reach the Kubernetes API
endpoint of the remote hosting cluster.

Do not use:

```yaml
server: https://kubernetes.default.svc
```

for this topology, because that points to Argo CD's own Kubernetes cluster.

The revised ApplicationSet uses:

```yaml
destination:
  name: hosting-cluster
  namespace: "{{ .namespace }}"
```

so the remote cluster is referenced by the name under which it was registered
in Argo CD.
