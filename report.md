# DSO202 Practical 4: GitOps with Argo CD on kind

**Name:** Tandin Zangmo | **Repo:** https://github.com/TandinZangmo456/dso202-gitops-demo | **Date:** 3 November 2026

## 1. Aim

The aim of this practical is to install Argo CD on a local kind cluster and use it to deploy a small web app straight from a Git repository. Then I test three GitOps behaviours: changing the app through Git, self-healing after a manual change, and pruning a resource that was deleted from Git.

## 2. Theory

**GitOps** means the Git repository is the single source of truth for what should run in the cluster. We do not run `kubectl apply` by hand. We change files in Git, and **Argo CD**, running inside the cluster, makes the cluster match Git. It keeps watching the repo and compares the *desired state* (YAML in Git) with the *live state* (what is running in Kubernetes).

- **Synced / OutOfSync** tells us if the live state matches Git.
- **Healthy** tells us if the resources are actually working.
- **Automated sync** applies new commits without a manual click.
- **selfHeal** puts things back if someone changes the cluster by hand.
- **prune** deletes resources from the cluster when they are deleted from Git.

**kind** (Kubernetes in Docker) runs each Kubernetes node as a Docker container. I used 1 control plane and 2 workers.

## 3. Setup and Argo CD installation

### 3.1 Fresh kind cluster

I deleted the old `dso202` cluster and created it again from `kind-config.yaml`. I also deleted another old cluster called `dso202-a2` to free memory (see Problems).
```bash
kind delete cluster --name dso202
kind create cluster --name dso202 --config kind-config.yaml
```
![kind cluster with 3 nodes](screenshots/01-kind-cluster-nodes.png)

### 3.2 Preparing install.yaml

My network is slow, so I did not want Kubernetes to download the Argo CD images again. I downloaded the stable `install.yaml` and changed `imagePullPolicy: Always` to `IfNotPresent` (8 places). Then I checked that the images in the file match my `argocd-images.txt`.
```bash
curl -sSL -o install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sed -i 's/imagePullPolicy: Always/imagePullPolicy: IfNotPresent/' install.yaml
```
![images in install.yaml match argocd-images.txt](screenshots/02-install-yaml-images-match.png)

The three images were `quay.io/argoproj/argocd:v3.5.7`, `ghcr.io/dexidp/dex:v2.45.1` and `public.ecr.aws/docker/library/redis:8.2.3-alpine`. They were already in Docker, so nothing was downloaded.

### 3.3 Loading images into the nodes

`kind load` did not work for me, so I imported the images into every node with `ctr`:
```bash
for node in $(kind get nodes --name dso202); do
  while read -r img; do
    docker save "$img" | docker exec -i "$node" ctr --namespace=k8s.io \
      images import --digests --snapshotter=overlayfs -
  done < argocd-images.txt
done
```
![images present on all 3 nodes](screenshots/05-images-imported-on-nodes.png)

### 3.4 Installing Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f install.yaml
```

![Argo CD install output](screenshots/06-argocd-install.png)
After about 50 seconds all 7 pods were Running and 1/1. The application controller had 0 restarts this time.
![7 Argo CD pods running](screenshots/07-argocd-pods-running.png)

### 3.5 Logging in to the UI

I read the admin password from the `argocd-initial-admin-secret` secret (it is hidden in the screenshot) and port-forwarded the server.
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

![hidden password and port-forward](screenshots/08-admin-password-port-forward.png)
Then I opened https://localhost:8080 and logged in as `admin`.
![Argo CD login page](screenshots/09-argocd-login.png)
![Argo CD after login](screenshots/10-argocd-logged-in.png)

## 4. Git repository and Application

I created the public repo `dso202-gitops-demo` with a README and added two manifests in the `manifests/` folder:

- `deployment.yaml`: `webapp-deploy`, 2 replicas, label `app: webapp`, image `nginxdemos/hello:plain-text`, port 80
- `service.yaml`: `webapp-svc`, type ClusterIP, port 80

![manifests](screenshots/03-manifests.png)
![commit and push](screenshots/04-git-push-manifests.png)

Next I wrote `argocd-app.yaml` outside the repo. The important part is the sync policy:
```yaml
  source:
    repoURL: https://github.com/TandinZangmo456/dso202-gitops-demo.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

![applying the Application](screenshots/11-apply-application.png)
About 40 seconds later (the nodes had to pull the nginx image) the app was **Synced** and **Healthy** with **2 pods**, and the `demo-app` namespace was created automatically.

![Synced, Healthy, 2 pods](screenshots/12-app-synced-healthy-2-pods.png)
![app tree in the UI](screenshots/13-ui-app-tree-2-pods.png)

## 5. Experiments

### Experiment 1: Change replicas through Git

I changed `replicas: 2` to `replicas: 4`, committed and pushed. To skip the 3 minute poll, I refreshed with `kubectl -n argocd annotate application my-webapp argocd.argoproj.io/refresh=normal --overwrite`.

![git diff and push for 4 replicas](screenshots/14-exp1-git-change-replicas-4.png)

**Result:** Argo CD synced to commit `6e155dd` and there were 4 pods, 2 on each worker.
![4 pods in terminal](screenshots/15-exp1-4-pods.png)
![4 pods in UI](screenshots/16-exp1-ui-4-pods.png)

### Experiment 2: Manual change and self-heal

I changed the cluster by hand, which is against GitOps:
```bash
kubectl scale deployment webapp-deploy -n demo-app --replicas=1
```
**Result:** within about 3 seconds `spec.replicas` was back to 4. The ready count dropped to 1 for a moment and then went back to 4 with 3 new pods.
![self-heal from 1 back to 4](screenshots/17-exp2-self-heal-terminal.png)
The UI shows a new sync at 15:52:33 started by the automated policy, still on the same Git commit.

![UI after self-heal](screenshots/18-exp2-ui-back-to-4.png)

### Experiment 3: Prune and revert

I removed the service from Git with `git rm manifests/service.yaml`, then committed and pushed.

**Result:** after a refresh the service was deleted from the cluster and the app stayed Synced and Healthy.
![service pruned in terminal](screenshots/19-exp3-service-pruned.png)
![service gone from UI](screenshots/20-exp3-ui-service-pruned.png)

Then I undid it with `git revert --no-edit HEAD` and pushed. **Result:** `webapp-svc` came back with a new ClusterIP and the deployment still had 4/4 pods.
![service back after revert](screenshots/21-exp3-revert-service-back.png)
![service back in UI](screenshots/22-exp3-ui-service-back.png)
The history page shows each deploy was started by the automated sync policy and took about 1 second.
![sync history](screenshots/23-ui-history.png)

## 6. Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| `kind load docker-image` failed with a content digest error | kind's loader had trouble with these image digests from Docker Desktop | Used `docker save` piped into `ctr images import --digests --snapshotter=overlayfs` on each node |
| Images would be downloaded again on a slow network | `install.yaml` uses `imagePullPolicy: Always` | Changed it to `IfNotPresent` so the imported images are used |
| Application controller OOMKilled in my earlier attempt | Docker Desktop only gives its VM about 1.7 GiB of RAM, and a second kind cluster `dso202-a2` was also running in it. Host swap was also full. | Deleted the extra `dso202-a2` cluster before installing. This time the controller had 0 restarts and the 3 nodes used about 1.2 GiB together |
| Dex pod restarted 2 times at start | It starts before the Argo CD secret is ready | No fix needed, it became 1/1 by itself |

## 7. Conclusion

In this practical I installed Argo CD on a 3 node kind cluster without downloading any images, and deployed a web app from my GitHub repo. All three experiments worked:

1. Changing replicas in Git changed the cluster to 4 pods.
2. A manual `kubectl scale` was undone in about 3 seconds by selfHeal.
3. Deleting the service in Git removed it from the cluster, and `git revert` brought it back.

This showed me that with GitOps, Git really is the source of truth. Every change has a commit, so it is easy to see who changed what and to roll back. I also learned that Argo CD needs a fair amount of memory, so on a small machine it is important to stop other clusters first.
