# Practical 4: GitOps with Argo CD


**Aim:** Deploy an app to Kubernetes with GitOps. Argo CD pulls the desired state from GitHub and keeps the cluster matching it. This report shows Git-driven scaling, self-healing and pruning.

---

## 1. Check the cluster

The lab uses the existing 3-node Kind cluster `dso202` (1 control plane, 2 workers). All nodes are Ready.

```bash
kubectl get nodes
```

![kubectl get nodes](screenshots/s1_nodes.png)

## 2. Install Argo CD

Argo CD was installed into its own `argocd` namespace. All 7 pods are Running.

```bash
kubectl get pods -n argocd
```

![kubectl get pods -n argocd](screenshots/s2_argocd_pods.png)

## 3. Log in to the Argo CD UI

The admin password was read from the `argocd-initial-admin-secret` secret (masked here). The UI was opened at https://localhost:8080 using `kubectl port-forward`.

![Initial admin password (masked)](screenshots/s3_password.png)

![Argo CD login page](screenshots/u1_login.png)

![Logged in, empty Applications page](screenshots/u1_login_apps.png)

## 4. Put the desired state in GitHub

The two manifests (`deployment.yaml` with 2 replicas, and `service.yaml`) were put in a `manifests/` folder of my repo `DSO202-practical-04` and pushed to `main`.

![The manifest files](screenshots/s4_manifests.png)

![git push to GitHub](screenshots/s5_push.png)

## 5. Register the application in Argo CD

A declarative Application (`argocd-app.yaml`) points Argo CD at the `manifests` folder, with automated sync, prune and self-heal turned on. Argo CD created the `demo-app` namespace and deployed 2 pods, one on each worker node.

![Applying the Application, then checking sync and pods](screenshots/s6_apply_app.png)

![Argo CD tree: Application, Deployment, ReplicaSet, 2 Pods and the Service](screenshots/u2_tree_2pods.png)

## 6. Experiment 1: Git is the source of truth

I changed `replicas: 2` to `replicas: 4` in Git and pushed. After a Refresh, Argo CD synced to the new commit and started 2 more pods. No `kubectl` was used to deploy.

![The change and the push](screenshots/e1_scale_push.png)

![Argo CD now shows 4 pods](screenshots/e1_tree_4pods.png)

![kubectl confirms 4 pods](screenshots/e1_pods4.png)

## 7. Experiment 2: Configuration drift and self-heal

I acted as a rogue admin and ran `kubectl scale ... --replicas=1`. Because `selfHeal: true`, Argo CD noticed the drift and restored 4 replicas within seconds (the 3 new pods are only 9s old).

![Manual scale to 1, then Argo CD restores 4](screenshots/e2_selfheal.png)

![Argo CD tree after self-heal](screenshots/e2_tree_healed.png)

## 8. Experiment 3: Automatic pruning and revert

I deleted `service.yaml` from Git. With `prune: true`, Argo CD deleted the `webapp-svc` Service from the cluster. Then `git revert` brought the file back, and Argo CD recreated the Service.

![Before: the Service exists](screenshots/e3_before_prune.png)

![Delete service.yaml from Git and push](screenshots/e3_prune_push.png)

![Argo CD tree: the Service is gone](screenshots/e3_tree_pruned.png)

![kubectl: no Service in demo-app](screenshots/e3_after_prune.png)

![git revert and push](screenshots/e3_revert.png)

![Argo CD tree: the Service is back](screenshots/e3_tree_restored.png)

![kubectl: Service recreated](screenshots/e3_restored.png)

## Conclusion

Git was the single source of truth. Every change (scale, delete, revert) was a Git commit, and Argo CD applied it automatically. A manual change made with `kubectl` was undone by self-heal, and a file deleted from Git was pruned from the cluster.
