# Runbook: Recreate Cluster & ArgoCD After Teardown

Use this whenever the EKS cluster has been deleted (to save cost) and you want
to bring the whole pipeline back up. Everything in Git/DockerHub already
survives teardown — this runbook only rebuilds the *compute* pieces.

---

## 0. Before you start

- [ ] Confirm `eks-cluster.yaml` still exists locally (project root)
- [ ] Confirm AWS CLI still authenticated: `aws sts get-caller-identity`

---

## 1. Start the EC2 VMs

AWS Console → EC2 → select `jenkins-server` → **Instance state → Start instance**
Repeat for `sonarqube-server`.

Wait for both to show **Running**.

⚠️ Public IPs will likely have changed (no Elastic IP configured).

---

## 2. Reconnect via PuTTY

- Copy new public IPs from EC2 console for both instances
- PuTTY → Host: `ubuntu@<new-ip>` → same `devops-learning-key.ppk`
- Confirm login as `ubuntu` (not root)

---

## 3. Recreate the EKS cluster

From local machine, in the folder containing `eks-cluster.yaml`:

```bash
eksctl create cluster -f eks-cluster.yaml
```

**DO NOT Ctrl+C this command.** Let it run fully to completion
(~15-20 minutes). It creates the control plane, addons, AND the nodegroup
together since both are defined in the config file.

---

## 4. Point kubectl at the new cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name flask-calculator-cluster
kubectl get nodes
```

Expect: 2 nodes, both `Ready`.

If `kubectl get nodes` shows a connection error to `127.0.0.1`, check
`kubectl config get-contexts` — make sure the EKS context (not `minikube`)
has the `*`.

---

## 5. Reinstall ArgoCD

ArgoCD lived inside the old cluster, so it was deleted along with it.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

Wait until all pods show `1/1 Running` (a couple of minutes).

---

## 6. Get admin password and log in

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
```

Decode (PowerShell):

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("PASTE_ENCODED_STRING_HERE"))
```

Port-forward (leave running in its own terminal):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Browse to `https://localhost:8080` → login `admin` / decoded password.

> If login fails with "invalid username or password" after 5 attempts,
> it's a lockout, not a wrong password. Fix via:
> ```bash
> kubectl -n argocd edit secret argocd-secret
> ```
> Delete the `admin.password` and `admin.passwordMtime` lines, save, then:
> ```bash
> kubectl -n argocd rollout restart deployment argocd-server
> ```
> Re-fetch the password and try again.

---

## 7. Recreate the ArgoCD Application

In the ArgoCD UI → **+ NEW APP**:

| Field | Value |
|---|---|
| Application Name | `flask-calculator` |
| Project | `default` |
| Sync Policy | **Automatic** (already proven working — no need for Manual-first this time) |
| Repository URL | `https://github.com/crazyfrog46/flask-calculator-manifests.git` |
| Revision | `HEAD` |
| Path | `.` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default`  ⚠️ check spelling carefully, not "defaullt" |

Click **Create**. With Automatic policy, it should sync itself within seconds.

---

## 8. Verify everything is live

```bash
kubectl get pods -n default
kubectl get deployments -n default
kubectl get services -n default
```

Copy the `EXTERNAL-IP` value from the Service and open it in a browser:

```
http://<external-ip-value>
```

Confirm the calculator works.

---

## 9. Confirm the full CI/CD loop still works

Make a trivial code change to the Flask app, push it, and confirm:

- Jenkins build triggers automatically (webhook)
- Tests, SonarQube, Docker build/push all succeed
- `Update Manifests Repo` stage commits a new tag
- ArgoCD auto-syncs the new image (check `kubectl get deployment ... -o jsonpath="{.spec.template.spec.containers[0].image}"`)

---

## Reminder: cost discipline

When done experimenting for the day:

```bash
eksctl delete cluster --name flask-calculator-cluster --region us-east-1
```

Then stop (not terminate) both EC2 instances from the console.

Nothing in GitHub or DockerHub needs any action — those are free and
permanent regardless of cluster state.
