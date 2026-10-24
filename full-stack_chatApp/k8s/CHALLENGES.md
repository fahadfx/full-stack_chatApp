# Kubernetes Deployment: Challenges and Fixes

This file records the problems I hit while deploying the full-stack chat app to Kubernetes, how I found the cause of each one, and how I fixed it.

## Setup

| | |
|---|---|
| **Cluster** | minikube (single node, containerd runtime) |
| **Namespace** | `chat-app` |
| **Frontend** | `fahadfx/chatapp-frontend` (React app served by nginx), 3 replicas, port 80 |
| **Backend** | `fahadfx/chatapp-backend` (Node.js + Express + Socket.io), 3 replicas, port 5001 |
| **Database** | `mongo:7.0`, 1 replica, port 27017, data stored on a PersistentVolumeClaim |

How a request flows:

```
Browser ──► frontend Service ──► frontend Pods (nginx)
                                      │
                                      ├── /            → static React files
                                      ├── /api/        → backend:5001
                                      └── /socket.io/  → backend:5001
                                                            │
                                            backend Service ──► backend Pods
                                                                   │
                                                   mongodb Service ──► MongoDB Pod ──► PVC
```

Only the frontend needs to be reachable from outside the cluster. nginx passes API and WebSocket traffic to the backend, so the backend and MongoDB stay internal.

## Summary

| # | Challenge | Symptom | Root cause | Fix |
|---|---|---|---|---|
| 1 | MongoDB would not start | `Error` / `CrashLoopBackOff` | Unpinned `mongo` image pulled MongoDB 9, which refuses to run on Linux kernel 6.19+ | Pinned `mongo:7.0` |
| 2 | Backend could not reach MongoDB | Connection errors in backend logs | No Service for MongoDB, so the name `mongodb` didn't resolve | Added `mongodb-service.yml` |
| 3 | Frontend pods would not start | `ErrImagePull` / `ImagePullBackOff` | The frontend image was built locally but never pushed to Docker Hub | `docker push fahadfx/chatapp-frontend:latest` |
| 4 | Could not open the app in a browser | No URL to visit | All Services are `ClusterIP`, which is internal only | `kubectl port-forward` to the frontend Service |
| 5 | Signup / login returned "Internal Server Error" | HTTP 500 | Env var was named `JWT_SECRETS`, but the code reads `JWT_SECRET` | Renamed the variable in the backend Deployment |

---

## 1. MongoDB pod in CrashLoopBackOff

**Symptom.** The MongoDB pod showed `0/1 Error` and restarted 6 times in about 7 minutes. The PVC was `Bound` and there were no storage events.

**Diagnosis.**

```bash
kubectl get pods -n chat-app
kubectl describe pod -n chat-app -l app=mongodb   # container starts, then backs off
kubectl logs -n chat-app deploy/mongodb-deployment
```

The log had one fatal line:

```
MongoDB cannot start: Linux kernel versions 6.19 and newer has a known incompatibility
with this version of MongoDB. See https://jira.mongodb.org/browse/SERVER-121912
```

**Root cause.** The Deployment used `image: mongo`, which means `mongo:latest`. That had become MongoDB 9.0.2. Containers share the node's kernel, and this machine runs kernel 7.0. MongoDB 8.0 and newer exit at startup on kernel 6.19 or newer.

**Fix.** In `mongodb-deployement.yml`:

```diff
-          image: mongo
+          image: mongo:7.0
```

```bash
kubectl apply -f mongodb-deployement.yml
```

**Lesson.** Always pin image tags. With `latest`, a deploy that worked yesterday can break today when a new major version is published. A `Bound` PVC plus a container that starts and then dies points at the application or the image, not at storage.

The full root-cause analysis is in [RCA-mongodb-pod-failure.md](RCA-mongodb-pod-failure.md).

---

## 2. Backend could not reach MongoDB

**Symptom.** MongoDB was running, but the backend could not connect to it.

**Root cause.** The backend connects with `mongodb://...@mongodb:27017/...`. The hostname `mongodb` only exists if a **Service** named `mongodb` exists in the same namespace. There was no such Service. Pod IPs change every time a pod restarts, so pods should always talk to each other through a Service name, never a pod IP.

**Fix.** Added `mongodb-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb          # this becomes the DNS name the backend uses
  namespace: chat-app
spec:
  selector:
    app: mongodb         # sends traffic to pods with this label
  ports:
  - port: 27017
    targetPort: 27017
```

```bash
kubectl apply -f mongodb-service.yml
```

The backend logs then showed `MongoDB connected: mongodb`.

**Lesson.** A Service's `metadata.name` is the hostname other pods use. Its `selector` must match the pod's labels. If `kubectl get endpoints <service> -n chat-app` is empty, the selector doesn't match any pods.

---

## 3. Frontend pods stuck in ImagePullBackOff

**Symptom.** The frontend pods never started. `kubectl describe pod` showed that Docker Hub refused the pull because the repository could not be accessed.

**Diagnosis.**

```bash
kubectl describe pod -n chat-app -l app=frontend   # Events: failed to pull image
docker images | grep chatapp                        # image exists locally
```

```
fahadfx/chatapp-backend:latest    c6eddf3e9df2   166MB
fahadfx/chatapp-frontend:latest   8d9aae050743   67.8MB
```

**Root cause.** The image had been built on my machine but **never pushed to Docker Hub**. Kubernetes does not use my laptop's Docker images. The kubelet on the node asks the container runtime for the image, and the runtime pulls it from the registry (Docker Hub). An image that only exists locally can't be pulled.

**Fix.**

```bash
docker login
docker push fahadfx/chatapp-frontend:latest
docker pull fahadfx/chatapp-frontend:latest              # check that Docker Hub now serves it
kubectl delete pods -n chat-app -l app=frontend         # the Deployment recreates them
kubectl get pods -n chat-app -w
```

**Lesson.** `ErrImagePull` / `ImagePullBackOff` means the node can't fetch the image. Check, in order: does the image exist in the registry, is the name and tag spelled exactly right, and does the registry need credentials (an `imagePullSecret`)? On minikube, `minikube image load` is a shortcut, but pushing to a registry is how it works on a real cluster (kubeadm, EKS, GKE, AKS).

---

## 4. Accessing the application

**Symptom.** All pods were `Running`, but there was no URL to open.

**Root cause.** The frontend Service is type `ClusterIP`, so it only has an IP inside the cluster.

```
service/frontend   ClusterIP   10.100.231.88   <none>   80/TCP
```

**Fix.** Forward a port from my machine to the Service:

```bash
kubectl port-forward -n chat-app svc/frontend 8080:80
```

```
kubectl port-forward  -n chat-app  svc/frontend  8080:80
                      │            │             │    └─ the Service's port
                      │            │             └────── the port on my machine
                      │            └──────────────────── what to forward to (<type>/<name>)
                      └───────────────────────────────── the namespace
```

Then open http://localhost:8080.

Use port **8080** specifically. In production mode the backend only accepts browser requests from `http://localhost:8080` and `http://localhost` (CORS allowlist in `backend/src/index.js`).

**Other options.**

| Option | Command | When to use |
|---|---|---|
| Port-forward | `kubectl port-forward -n chat-app svc/frontend 8080:80` | Local testing, no YAML change |
| minikube helper | `minikube service frontend -n chat-app` | minikube only, needs a NodePort Service |
| NodePort | `type: NodePort` + `nodePort: 30080` in the Service | Reach the app on `<node-ip>:30080` |
| LoadBalancer / Ingress | `type: LoadBalancer`, or an Ingress controller | Real clusters (EKS, GKE, AKS) |

---

## 5. Signup and login returned HTTP 500

**Symptom.** The app loaded, but signing up failed with "Internal Server Error".

**Diagnosis.**

```bash
kubectl logs -n chat-app deploy/backend-deployment --tail=50
```

```
Error in signup controller secretOrPrivateKey must have a value
```

That error comes from the `jsonwebtoken` library when the secret used to sign the login token is empty.

```bash
grep -rn "process.env" backend/src
# backend/src/lib/utils.js:  jwt.sign({ userId }, process.env.JWT_SECRET, ...)

kubectl exec -n chat-app deploy/backend-deployment -- printenv
# JWT_SECRETS was set, JWT_SECRET was missing
```

**Root cause.** A typo. The Deployment set the variable as `JWT_SECRETS` (extra **S**), but the code reads `JWT_SECRET`. The Secret itself was correct. It was just exposed under the wrong name.

**Fix.** In `backend-deployement.yml`:

```diff
-            - name: JWT_SECRETS
+            - name: JWT_SECRET
               valueFrom:
                 secretKeyRef:
                   name: chatapp-secrets
                   key: jwt
```

```bash
kubectl apply -f backend-deployement.yml
kubectl rollout status -n chat-app deploy/backend-deployment
```

A test signup through the frontend then returned `201 Created` and set the login cookie.

**Lesson.** Kubernetes does not check that your env var names match what the application expects. `kubectl exec ... -- printenv` shows exactly what the container sees. Compare it with `grep process.env` in the source.

---

## Debugging cheat sheet

| Pod status | What it usually means | First command to run |
|---|---|---|
| `Pending` | Can't be scheduled (resources, unbound PVC, node selector) | `kubectl describe pod <pod> -n chat-app` (Events section) |
| `ContainerCreating` (stuck) | Volume mount or image problem | `kubectl describe pod <pod> -n chat-app` |
| `ErrImagePull` / `ImagePullBackOff` | Image missing from registry, wrong name, or no credentials | `kubectl describe pod <pod> -n chat-app` |
| `Error` / `CrashLoopBackOff` | Container started, then the app exited | `kubectl logs <pod> -n chat-app --previous` |
| `Running` but the app fails | Config, env vars, networking | `kubectl logs`, `kubectl exec -- printenv`, `kubectl get endpoints` |

```bash
kubectl get pods,svc,pvc -n chat-app                  # overview
kubectl describe pod <pod> -n chat-app                # events: scheduling, pulling, mounting
kubectl logs <pod> -n chat-app [--previous]           # application output
kubectl exec -n chat-app <pod> -- printenv            # env vars the container sees
kubectl get endpoints <service> -n chat-app           # does the Service select any pods?
kubectl rollout status deploy/<name> -n chat-app      # wait for an update to finish
```

## Known limitations and next steps

- [ ] **Credentials are in plain text.** The MongoDB root password is written directly in `mongodb-deployement.yml` and `backend-deployement.yml`. Move it into a Secret and reference it with `secretKeyRef`.
- [ ] **Secrets are only base64-encoded**, not encrypted. Anyone with the repo can decode `secrets.yml`. Use Sealed Secrets, External Secrets, or a cloud secret manager for anything real.
- [ ] **Images use `:latest`.** Tag and pin versions (e.g. `chatapp-backend:v1.0.0`) so rollouts and rollbacks are predictable.
- [ ] **`mongodb-pv.yml` is unused.** The PVC has no `storageClassName`, so minikube's default class provisions its own volume. Delete the manual PV, or set the same `storageClassName` on both.
- [ ] **Cloudinary isn't configured.** `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY` and `CLOUDINARY_API_SECRET` aren't set, so profile picture upload fails.
- [ ] **No health probes or resource limits.** Add `readinessProbe` / `livenessProbe` (the backend has a `/health` route) and CPU/memory `requests` / `limits`.
- [ ] **No Ingress.** Replace port-forward with an Ingress controller for a production-style setup.
