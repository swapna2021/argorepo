 
# React → Kubernetes → Argo CD (Docker Desktop) — End‑to‑End Guide

This guide turns a simple **Create React App (CRA)** into a **Kubernetes deployment** managed by **Argo CD**, using **Docker Desktop with Kubernetes**. It’s written for doing everything from a **Docker tools container** (for `kubectl`/`argocd`) while keeping **Docker builds on the host**.

---

## 0) What you’ll need (prereqs)

- **Docker Desktop** with **Kubernetes** enabled.
- **kubectl** context set to `docker-desktop` (or mount your `~/.kube/config` into a tools container).
- A **Docker Hub account** (or any container registry).
- A **GitHub repo** (public or private). Example used below: `https://github.com/<you>/argorepo.git`.
- For **private** GitHub repos: a **Personal Access Token (PAT)** with `repo` (read) scope.

---

## 1) Create the React app

```bash
npx create-react-app hello-app
cd hello-app
```

Edit `src/App.js` with a simple Hello message (optional):

```javascript
function App() {
  return <h1>Hello, World!</h1>;
}
export default App;
```

---

## 2) Add a production Dockerfile

Create `Dockerfile` in the project root:

```dockerfile
# Build stage
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and push **on the host** (not in the container):

```bash
# from the hello-app folder on your host
docker build -t <dockerhub-username>/argohello:v1 .
docker push <dockerhub-username>/argohello:v1
```

> Replace `<dockerhub-username>` with your Docker Hub username. Make sure the image/tag exists and is **public** (or plan to use an imagePullSecret later).

---

## 3) Kubernetes manifests (plain YAML)

Create a folder `k8s/` with two files:

**`k8s/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-react
  labels:
    app: hello-react
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-react
  template:
    metadata:
      labels:
        app: hello-react
    spec:
      containers:
      - name: hello-react
        image: <dockerhub-username>/argohello:v12
        ports:
        - containerPort: 80
# If the image is private, add:
#      imagePullSecrets:
#      - name: regcred
```

**`k8s/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-react-service
spec:
  type: ClusterIP   # use NodePort if you want permanent external access
  selector:
    app: hello-react
  ports:
  - port: 80
    targetPort: 80
```

> For **NodePort**, switch `type: NodePort` and optionally set `nodePort: 30080`.

---

## 4) Put the manifests in Git

Create a GitHub repository and push the files, e.g.:

```
argorepo/
└── ArgoDemo/
    └── hello-app/
        ├── k8s/
        │   ├── deployment.yaml
        │   └── service.yaml
        ├── Dockerfile         (optional to include)
        └── src/...            (optional to include)
```

```bash
git init
git remote add origin https://github.com/<you>/argorepo.git
git checkout -b master
git add .
git commit -m "React + K8s manifests"
git push -u origin master
```

> If you keep the repo **private**, Argo CD will need credentials. If you make it **public**, it can be used without credentials.

---

## 5) Start a tools container (kubectl + argocd inside Docker)

Run this **on your host** to launch a container that can talk to your cluster by mounting your kubeconfig:

```bash
docker run -it --name kube-tools --rm \
  -v $HOME/.kube:/root/.kube \
  -p 8080:8080 \
  alpine:3.20 sh
```

Inside the container, install tools and verify connectivity:

```sh
apk add --no-cache curl bash tar gzip ca-certificates
KVER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
curl -L -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/${KVER}/bin/linux/amd64/kubectl
chmod +x /usr/local/bin/kubectl
curl -L -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

kubectl get nodes   # should show the docker-desktop node
```

---

## 6) Install Argo CD in the cluster

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait --for=condition=Available deploy/argocd-server --timeout=180s
```

Expose the UI:

**Option A: Port‑forward (quick)**
```sh
kubectl -n argocd port-forward svc/argocd-server 8080:443
# open https://localhost:8080  (proceed past the self-signed cert warning)
```

**Option B: NodePort (persistent)**
```sh
kubectl -n argocd patch svc argocd-server \
  -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8080,"nodePort":30080}]}}'
# open https://localhost:30080
```

Get initial admin password:
```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Login with **admin / <that password>** (UI) or:
```sh
argocd login localhost:8080 --username admin --password '<PASTE>' --insecure
```

---

## 7) Give Argo CD access to your repo

### If the repo is **public**: no action needed.

### If the repo is **private**: add it with a PAT
```sh
argocd repo add https://github.com/<you>/argorepo.git \
  --username <you> \
  --password <GITHUB_PAT>
```
> Create a **Personal Access Token (classic)** on GitHub with **repo** scope.

---

## 8) Create the Argo CD Application (GitOps)

Create `argo-app.yaml` (inside the tools container or on your host):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello-react-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/argorepo.git
    targetRevision: master
    path: ArgoDemo/hello-app/k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:
```sh
kubectl apply -f argo-app.yaml
kubectl get applications -n argocd
```

Open the Argo CD UI → you should see `hello-react-app` **Healthy** and **Synced**.

---

## 9) Access the React app

**If Service type is ClusterIP** (from this guide):
```sh
kubectl port-forward svc/hello-react 3400:80 -n default
# open http://localhost:3400
```

**If Service type is NodePort** (e.g., 30080):
```sh
kubectl get svc hello-react -n default
# open http://localhost:<nodePort>   e.g., http://localhost:30080
```

---

## 10) Update flow (how GitOps redeploys)

1. Change app code → build a **new image** and **push** a **new tag**:
   ```bash
   docker build -t <dockerhub-username>/argohello:v2 .
   docker push <dockerhub-username>/argohello:v2
   ```
2. Update `k8s/deployment.yaml` with the new image tag, commit & push:
   ```yaml
   image: <dockerhub-username>/argohello:v2
   ```
3. Argo CD detects the Git change and performs a rolling update automatically.

---

## 11) Troubleshooting

- **`ImagePullBackOff`**  
  - Ensure the exact image:tag exists and is public:  
    `docker pull <dockerhub-username>/argohello:v1`  
  - If private, create a pull secret and reference it:
    ```sh
    kubectl create secret docker-registry regcred \
      --docker-username='<dockerhub-username>' \
      --docker-password='<dockerhub-password>' \
      --docker-email='<email>'
    ```
    ```yaml
    # in deployment spec:
    spec:
      imagePullSecrets:
      - name: regcred
    ```

- **`repository not accessible` in Argo CD**  
  - Public: check the repo URL and path.
  - Private: add via CLI `argocd repo add ...` with a valid PAT, or make the repo public.

- **`kubectl 127.0.0.1:6443 refused` inside tools container**  
  - You didn’t mount kubeconfig. Recreate the container with `-v $HOME/.kube:/root/.kube`.

- **Chrome TLS warning**  
  - Argo CD uses a self‑signed cert. Click **Advanced → Proceed**.  
  - Or expose HTTP by patching the service to a plain HTTP port if you prefer.

---

## 12) Clean up

```bash
# remove app resources
kubectl delete -f argo-app.yaml

# or delete everything you applied manually
kubectl delete -f k8s/

# remove Argo CD
kubectl delete ns argocd
```

---

## 13) Optional: Reusable tools image

If you want a permanent Docker image with `kubectl` and `argocd` preinstalled, create this Dockerfile and build it once:

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache curl bash tar gzip ca-certificates
RUN KVER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt) && \
    curl -L -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/${KVER}/bin/linux/amd64/kubectl && \
    chmod +x /usr/local/bin/kubectl && \
    curl -L -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && \
    chmod +x /usr/local/bin/argocd
CMD ["sh"]
```

Run it with your kubeconfig mounted:
```bash
docker build -t kube-tools:latest .
docker run -it --rm -v $HOME/.kube:/root/.kube -p 8080:8080 kube-tools:latest
```

---

**You’re done!** You now have a full GitOps pipeline: React → Docker image → Kubernetes manifests in Git → Argo CD auto‑sync in Docker Desktop.
