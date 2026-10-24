# RCA: MongoDB pod failing to start on minikube

| | |
|---|---|
| **Date** | 2026-10-01 |
| **Environment** | Local minikube (single node, containerd), namespace `chat-app` |
| **Affected component** | `mongodb-deployment` (Deployment, 1 replica) |
| **Status** | Resolved |
| **Impact** | MongoDB unavailable; pod stuck in `Error` / `CrashLoopBackOff` with 6 restarts. No data loss (the data directory was empty). |

## Summary

The MongoDB pod crashed on every start because the Deployment used an unpinned image (`image: mongo`), which resolved to MongoDB 9.0.2. MongoDB 8.0 and newer refuse to start on Linux kernel 6.19 or newer, and the host (whose kernel minikube shares) runs kernel 7.0.0. Pinning the image to `mongo:7.0` fixed it.

The PersistentVolume and PersistentVolumeClaim were not the cause. The PVC was `Bound` and the volume mounted correctly.

## Symptoms

- `kubectl get pods -n chat-app` showed `mongodb-deployment-7646d876c4-r4wg6` at `0/1 Error`, 6 restarts in about 7 minutes.
- Pod events showed the image pulling and the container starting successfully each time, followed by `Back-off restarting failed container`.
- No `FailedMount`, `FailedScheduling` or unbound-PVC events.

## Root cause

The container log contained a single fatal line:

```
MongoDB cannot start: Linux kernel versions 6.19 and newer has a known incompatibility
with this version of MongoDB. See https://jira.mongodb.org/browse/SERVER-121912
```

Three facts combine to produce it:

1. **Unpinned image.** `image: mongo` is equivalent to `mongo:latest`. The image pulled was built on 2026-09-30 and contains MongoDB 9.0.2.
2. **Shared kernel.** Containers use the node's kernel, and minikube's node uses the host's. The node reports kernel `7.0.0-31-generic`.
3. **Known incompatibility.** SERVER-121912 ("Upgrade tcmalloc to include rseq bug fix") describes startup crashes in MongoDB 8.0+ on kernel 6.19, caused by the bundled TCMalloc violating the kernel's rseq ABI. MongoDB 9.0.2 detects the kernel version and exits at startup.

The ticket is closed as "Gone away" with no fix version listed, so it does not say which 8.x or 9.x release, if any, runs on this kernel.

## How it was diagnosed

| Step | Command | Finding |
|---|---|---|
| 1 | `kubectl get pods,pvc -A` and `kubectl get pv` | Pod in `Error`; PVC `Bound` to a dynamically provisioned volume |
| 2 | `kubectl -n chat-app describe pod -l app=mongodb` | Container starts, then backs off; no storage events |
| 3 | `kubectl -n chat-app logs deploy/mongodb-deployment` | Fatal kernel-incompatibility message |
| 4 | `crictl inspecti mongo:latest` on the node | Image is MongoDB 9.0.2 |
| 5 | `kubectl get node minikube` | Kernel 7.0.0-31-generic |
| 6 | Throwaway pod with `mongo:7.0` | Started cleanly, reported version 7.0.43 |

## Resolution

Changed one line in `mongodb-deployement.yml`:

```diff
-          image: mongo
+          image: mongo:7.0
```

Applied with `kubectl apply -f mongodb-deployement.yml`.

Verified after rollout:

- Pod `mongodb-deployment-69d45969c5-qv4cx` is `1/1 Running` with 0 restarts.
- `mongosh -u admin -p admin123` authenticates and returns version `7.0.43`.
- WiredTiger files are present in the mounted volume, so persistence is working.

The throwaway test pod was deleted.

## Contributing factors and other findings

None of these caused the crash, but they were found during the review.

| # | Finding | File | Recommendation |
|---|---|---|---|
| 1 | The hand-written PV is unused. The PVC has no `storageClassName`, so minikube's default `standard` class provisioned its own volume and `mongodb-pv` stays `Available`. | `mongodb-pv.yml`, `mongodb-pvc.yml` | Delete `mongodb-pv.yml` and rely on dynamic provisioning, or set the same `storageClassName` (e.g. `manual`) on both PV and PVC. |
| 2 | `namespace: chat-app` on the PV is ignored; PersistentVolumes are cluster-scoped. | `mongodb-pv.yml` | Remove the field. |
| 3 | `hostPath: /data` is too broad for a database volume. | `mongodb-pv.yml` | Use a dedicated path such as `/data/mongodb`. |
| 4 | No Service exists in `chat-app`, so the backend cannot reach MongoDB by DNS name. | (missing) | Add a ClusterIP Service on port 27017 selecting `app: mongodb`. |
| 5 | Root password is in plain text in the Deployment. | `mongodb-deployement.yml` | Move it to a Secret and use `secretKeyRef`. |
| 6 | A `Released` volume, `pvc-401030f4-14cd-49b5-9e77-7190bb57c83b`, is left over from an earlier attempt. | cluster | `kubectl delete pv pvc-401030f4-14cd-49b5-9e77-7190bb57c83b` |
| 7 | The backend image is also unpinned (`fahadfx/chatapp-backend:latest`). | `backend-deployemnt.yml` | Tag and pin a specific version. |

## Prevention

1. **Pin every image tag.** A bare image name or `latest` means a deploy that worked yesterday can break today when a new major version is published. Pin at least the major.minor (`mongo:7.0`).
2. **Read the container logs first for `Error` and `CrashLoopBackOff`.** These statuses mean the container started and then exited, so the application log has the reason:
   ```
   kubectl -n chat-app logs <pod>
   kubectl -n chat-app logs <pod> --previous
   kubectl -n chat-app describe pod <pod>
   ```
3. **Use pod status to narrow the search.** Storage problems appear as `Pending` or `ContainerCreating` with `FailedMount` or unbound-PVC events. A `Bound` PVC plus a container that starts and dies points at the application or image.
4. **Check kernel compatibility before upgrading MongoDB.** Any move to MongoDB 8+ on this machine needs a release confirmed to run on kernel 6.19+. Test it in a throwaway pod first.

## Follow-up actions

- [ ] Add a MongoDB Service
- [ ] Move MongoDB credentials to a Secret
- [ ] Remove or correctly wire `mongodb-pv.yml`
- [ ] Delete the leftover `Released` PV
- [ ] Pin the backend and frontend image tags
