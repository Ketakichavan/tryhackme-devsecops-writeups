# Kubernetes — TryHackMe DevSecOps Pathway

Writeup covering Kubernetes fundamentals, hands-on cluster interaction with `kubectl`, and a practical walkthrough of Kubernetes security hardening using RBAC (Role-Based Access Control).

**Environment:** Browser-based AttackBox, Minikube cluster

---

## Objective

Deploy a simple nginx web application onto a Minikube cluster, find and exploit a Kubernetes Secret to retrieve a flag, then fix the exposure by setting up RBAC so only the right identity can access that Secret going forward.

---

## Phase One: Explore — Deploying to the Cluster

The Minikube cluster is started first, and a quick check confirms the default control plane and worker node components are running:

```bash
minikube start
kubectl get pods -A
```

This shows the standard `kube-system` pods — `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `coredns`, and `storage-provisioner` — confirming the control plane and worker node processes are healthy.

Two configuration files are provided:

- `nginx-deployment.yaml` — defines a Deployment with a single nginx pod replica
- `nginx-service.yaml` — defines a `NodePort` Service, exposing the deployment on port `8080` and targeting container port `80`

The Service is applied before the Deployment, which is standard practice in Kubernetes since services populate environment variables in containers at startup:

```bash
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-deployment.yaml
```

A follow-up check confirms the pod is running:

```bash
kubectl get pods -A
```

<img width="1301" height="650" alt="01-deployment-and-pods-running" src="https://github.com/user-attachments/assets/cd6e8681-77c2-4ecf-a445-5514d8aaebfd" />


---

## Phase Two: Interact — Finding and Exploiting the Secret

`kubectl port-forward` tunnels the Kubernetes service port to a local port so it's reachable from the browser:

```bash
kubectl port-forward service/nginx-service 8090:8080
```

<img width="1140" height="125" alt="02-port-forward" src="https://github.com/user-attachments/assets/ea66a658-2345-464a-9b0e-e30bee9e3f18" />


Navigating to `http://localhost:8090/` brings up a themed login page — "Jurassic Land Terminal Access" — asking for credentials.

<img width="1302" height="815" alt="03-login-page-empty" src="https://github.com/user-attachments/assets/32e13608-de17-4863-b188-ea29ce7b6598" />


A check for Kubernetes Secrets in the default namespace turns up something useful:

```bash
kubectl get secrets
kubectl describe secret terminal-creds
```

A secret named `terminal-creds` is found, containing `username` and `password` fields. Kubernetes Secrets are base64-encoded, not encrypted, by default, so both values are decoded:

```bash
kubectl get secret terminal-creds -o jsonpath='{.data.username}' | base64 --decode
kubectl get secret terminal-creds -o jsonpath='{.data.password}' | base64 --decode
```

<img width="1302" height="721" alt="04-secret-discovery-decode-REDACTED" src="https://github.com/user-attachments/assets/9eb5425e-6b1e-4875-b5d5-b254cb3b9e69" />



The decoded credentials are used to log into the terminal application, and the flag is retrieved.

<img width="1302" height="656" alt="05-login-submitted-REDACTED" src="https://github.com/user-attachments/assets/0c99e1eb-36e5-43e8-bba9-0e86c3ba0bbd" />


**Flag:** `THM{REDACTED}`

---

## Phase Three: Secure — Fixing It with RBAC

Now that the secret has been exposed, the next step is locking it down with Role-Based Access Control, so only an authorized identity can read it.

Two service accounts are created — one standard, non-privileged identity, and one admin identity that legitimately needs access:

```bash
kubectl create sa terminal-user
kubectl create sa terminal-admin
```

Two configuration files handle the actual restriction:

**`role.yaml`** defines a Role (`secret-admin`) scoped to only the `get` verb on the specific secret `terminal-creds` — not a blanket grant across all secrets:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: secret-admin
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["terminal-creds"]
```

**`role-binding.yaml`** binds that Role to the `terminal-admin` service account only:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-admin-binder
  namespace: default
subjects:
- kind: ServiceAccount
  name: terminal-admin
  namespace: default
roleRef:
  kind: Role
  name: secret-admin
  apiGroup: rbac.authorization.k8s.io
```

Both are applied:

```bash
kubectl apply -f role.yaml
kubectl apply -f role-binding.yaml
```

<img width="900" height="390" alt="06-rbac-role-config-and-sa-creation" src="https://github.com/user-attachments/assets/3fb7f48b-0f85-4f38-93fb-f33a8cc2d626" />


### Verification

`kubectl auth can-i` confirms whether the restriction is actually working:

```bash
kubectl auth can-i get secret/terminal-creds --as=system:serviceaccount:default:terminal-user
# → no

kubectl auth can-i get secret/terminal-creds --as=system:serviceaccount:default:terminal-admin
# → yes
```

`terminal-user` is correctly denied, while `terminal-admin` keeps the access it needs — confirming least-privilege access is now in place on the `terminal-creds` secret.

<img width="1305" height="570" alt="07-rbac-apply-and-verification" src="https://github.com/user-attachments/assets/432faab8-c050-425a-99ab-e526db72ac86" />


---

## Key Takeaways

- Kubernetes Secrets are base64-encoded, not encrypted, by default. Anyone with `get` access to a secret can decode it in seconds, so encryption at rest and tight RBAC scoping both matter.
- Least privilege through RBAC doesn't take much effort — a Role can be scoped down to a single verb (`get`) on a single named resource (`resourceNames: ["terminal-creds"]`) instead of granting broad access.
- Service accounts are how pods and applications get an identity when interacting with the Kubernetes API. Separating "admin" and "standard" identities is a simple, practical first step toward least-privilege design.
- `kubectl auth can-i` is a quick, reliable way to check that an RBAC policy is actually behaving as intended before trusting it in a real environment.

---

## Tools Used

`minikube`, `kubectl` (apply, get, describe, logs, port-forward, exec, auth can-i, create sa), `base64`

## Notes

Flag values and decoded credentials are redacted from this writeup and its screenshots in line with TryHackMe's platform policy on public content.
