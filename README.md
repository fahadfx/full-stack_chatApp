# Real-Time Chat App on Kubernetes

Deploying a full-stack real-time chat application (React, Node.js, Socket.io, MongoDB) to Kubernetes, and debugging the production-style failures that came up along the way.

> **Credit:** the application code (frontend, backend, Dockerfiles) comes from [iemafzalhassan/full-stack_chatApp](https://github.com/iemafzalhassan/full-stack_chatApp), under the MIT License. The Kubernetes deployment, troubleshooting and documentation in [`k8s/`](k8s/) are my work.

## What I built

- **Kubernetes manifests** for the whole stack in a dedicated `chat-app` namespace: Deployments, Services, a PersistentVolumeClaim for MongoDB data, and a Secret for the JWT signing key
- **Container images** pushed to Docker Hub (`fahadfx/chatapp-frontend`, `fahadfx/chatapp-backend`) so any cluster can pull them
- **Five real deployment failures diagnosed and fixed**, each written up with symptoms, the commands used to find the cause, the fix and the lesson: [k8s/CHALLENGES.md](k8s/CHALLENGES.md)
- **A root-cause analysis** of a MongoDB CrashLoopBackOff traced to a MongoDB / Linux kernel incompatibility: [k8s/RCA-mongodb-pod-failure.md](k8s/RCA-mongodb-pod-failure.md)

## Architecture

```
                         ┌──────────────── namespace: chat-app ─────────────────┐
                         │                                                      │
 Browser ── port-forward ┼─► frontend Service ──► frontend Pods ×3 (nginx)      │
   localhost:8080        │      ClusterIP :80          │                        │
                         │                             ├─ /            static React build
                         │                             ├─ /api/        ─┐       │
                         │                             └─ /socket.io/  ─┤       │
                         │                                              ▼       │
                         │                      backend Service ──► backend Pods ×3 (Node.js)
                         │                        ClusterIP :5001         │     │
                         │                                                │  JWT_SECRET from Secret
                         │                                                ▼     │
                         │                      mongodb Service ──► MongoDB Pod ×1 (mongo:7.0)
                         │                        ClusterIP :27017        │     │
                         │                                                ▼     │
                         │                                         PVC  mongodb-pvc (5Gi)
                         └──────────────────────────────────────────────────────┘
```

Only the frontend is exposed. Its nginx config passes `/api/` and `/socket.io/` to the `backend` Service, so the backend and database are never reachable from outside the cluster.

| Component | Image | Replicas | Port | Kubernetes objects |
|---|---|---|---|---|
| Frontend | `fahadfx/chatapp-frontend:latest` | 3 | 80 | Deployment, Service |
| Backend | `fahadfx/chatapp-backend:latest` | 3 | 5001 | Deployment, Service, Secret |
| Database | `mongo:7.0` | 1 | 27017 | Deployment, Service, PVC |

## Deploy it

**Requirements:** a Kubernetes cluster (I used minikube; kubeadm, EKS, GKE or AKS work the same way) and `kubectl`.

```bash
git clone https://github.com/fahadfx/full-stack_chatApp.git
cd full-stack_chatApp/k8s

kubectl apply -f namespace.yml          # create the namespace first
kubectl apply -f secrets.yml
kubectl apply -f mongodb-pvc.yml
kubectl apply -f mongodb-deployement.yml -f mongodb-service.yml
kubectl apply -f backend-deployement.yml -f backend-service.yml
kubectl apply -f frontend-deployement.yml -f frontend-service.yml

kubectl get pods -n chat-app -w         # wait until all 7 pods are 1/1 Running
```

Open the app:

```bash
kubectl port-forward -n chat-app svc/frontend 8080:80
```

Then go to **http://localhost:8080**. Use port 8080, because the backend's CORS allowlist only accepts `http://localhost:8080` and `http://localhost`.

Check that it works:

```bash
kubectl logs -n chat-app deploy/backend-deployment    # expect "MongoDB connected: mongodb"
```

## Challenges solved

| # | Problem | Root cause | Fix |
|---|---|---|---|
| 1 | MongoDB pod in `CrashLoopBackOff` | `mongo:latest` resolved to MongoDB 9, which refuses to start on Linux kernel 6.19+ | Pinned `mongo:7.0` ([RCA](k8s/RCA-mongodb-pod-failure.md)) |
| 2 | Backend couldn't connect to MongoDB | No Service named `mongodb`, so the hostname didn't resolve | Added a ClusterIP Service |
| 3 | Frontend pods in `ImagePullBackOff` | Image was built locally but never pushed to Docker Hub | `docker push` to the registry |
| 4 | No way to open the app | All Services are ClusterIP (internal only) | `kubectl port-forward` to the frontend Service |
| 5 | Signup returned HTTP 500 | Env var named `JWT_SECRETS`, but the code reads `JWT_SECRET` | Renamed the variable, found with `kubectl logs` and `printenv` |

Full write-ups, with commands and output: **[k8s/CHALLENGES.md](k8s/CHALLENGES.md)**

## What I learned

- **Pin image tags.** `latest` changed underneath me and broke a working database.
- **Pod status tells you where to look.** `ImagePullBackOff` means a registry problem, `CrashLoopBackOff` means read the app logs, and `Pending` means scheduling or storage.
- **Pods talk through Services, not IPs.** A Service's name is the DNS hostname, and its selector must match the pod labels.
- **Kubernetes doesn't check your config against your app.** A misspelled env var deploys fine and fails at runtime. `kubectl exec -- printenv` shows what the container really sees.
- **Kubernetes is declarative.** `kubectl apply` stores the desired state, and controllers keep reconciling the cluster towards it. That's why deleting a pod just gets it recreated.

## Next steps

- [ ] Move the MongoDB credentials out of the Deployment YAML and into a Secret
- [ ] Pin versioned image tags instead of `:latest`
- [ ] Add readiness and liveness probes (the backend has a `/health` route) and resource requests/limits
- [ ] Replace port-forward with an Ingress controller
- [ ] Deploy to a managed cluster (EKS) and add a CI/CD pipeline that builds, pushes and deploys the images
- [ ] Configure Cloudinary credentials so profile picture upload works

## Tech stack

- **Orchestration:** Kubernetes (minikube), kubectl
- **Containers:** Docker, Docker Hub, containerd
- **Application:** React, Vite, TailwindCSS, Zustand, Node.js, Express, Socket.io, JWT
- **Database:** MongoDB 7.0
- **Web server / proxy:** nginx

## Screenshots

![Chat](frontend/public/chat.png)

![Login](frontend/public/login.png)

![Settings](frontend/public/settings.png)

## License

MIT, see [LICENSE](LICENSE). The original application code is © its authors. See the credit at the top of this file.
