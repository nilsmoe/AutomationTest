# Fleet CD Test – NGINX

Test [Rancher Fleet](https://fleet.rancher.io/) continuous delivery from Git. Deploys a simple NGINX pod.

## What’s in this bundle

- **nginx-deployment.yaml** – Deployment + Service
- **nginx-ingress.yaml** – Optional Ingress for `nginx.int.cloudshift.no`
- **fleet.yaml** – Fleet bundle config (namespace: `fleet-test`)

## Setup Fleet GitRepo in Rancher

### Option A: Rancher UI

1. In Rancher, go to **Continuous Delivery** (Fleet).
2. Click **Add Git Repo**.
3. Use:
   - **Name:** `fleet-test-nginx`
   - **Repository URL:** `https://github.com/YOUR_ORG/cloudshift_it_lab` or your Azure DevOps URL
   - **Branch:** `devopsClient` (or your branch)
   - **Path:** `kubernetes/fleet-test`
   - **Target namespace:** `fleet-local` (single-cluster) or `fleet-default` (multi-cluster)

4. For a **private repo**, create a secret first:
   ```bash
   kubectl create secret generic git-auth -n fleet-local \
     --from-literal=username=YOUR_GITHUB_USER \
     --from-literal=password=YOUR_GITHUB_TOKEN
   ```
   Then choose **Private** and select that secret.

5. Click **Create**.

### Option B: kubectl (GitRepo YAML)

```bash
kubectl apply -f - <<EOF
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: fleet-test-nginx
  namespace: fleet-local
spec:
  repo: https://github.com/YOUR_ORG/cloudshift_it_lab
  branch: devopsClient
  paths:
    - kubernetes/fleet-test
  pollingInterval: 15s
EOF
```

For a private repo, add:

```yaml
spec:
  clientSecretName: git-auth  # Create secret first in fleet-local ns
```

## Verify

```bash
kubectl get pods -n fleet-test
kubectl get svc -n fleet-test
kubectl get ingress -n fleet-test  # if Ingress is applied
```

Fleet status in Rancher: **Continuous Delivery** → **fleet-test-nginx** → check **Ready** and **Resources**.

## Optional: Expose NGINX via Ingress

1. Add DNS in CoreDNS (`Docker/core-dns/int.cloudshift.no.zone`):
   ```
   nginx         IN      A       192.168.40.140
   ```

2. `nginx-ingress.yaml` is part of the bundle; Fleet will deploy it.

3. Visit https://nginx.int.cloudshift.no – you should see the NGINX welcome page.

## Test the GitOps flow

1. Edit `nginx-deployment.yaml` (e.g. change `replicas` to 2).
2. Commit and push to your branch.
3. Within ~15 seconds Fleet picks up the change and updates the cluster.
4. Confirm with `kubectl get pods -n fleet-test`.

## Troubleshooting

- **Bundle not ready:** Check Fleet logs: `kubectl logs -n fleet-system -l app=fleet-controller`
- **Git auth failed:** Ensure the secret exists and is referenced in the GitRepo
- **Wrong path:** Ensure `paths` includes `kubernetes/fleet-test` (or the correct path in your repo)
