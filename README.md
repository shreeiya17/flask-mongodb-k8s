# Flask and MongoDB on Kubernetes (Minikube)

A Python Flask application backed by an authenticated MongoDB, deployed on a local Minikube cluster with persistent storage, internal-only database access, CPU-based autoscaling, and explicit resource requests and limits.

Docker image: `docker.io/shreeiya/flask-mongo:v1` (public, multi-architecture: `linux/amd64` and `linux/arm64`)

## Contents

1. [Repository structure](#repository-structure)
2. [Requirements checklist](#requirements-checklist)
3. [Prerequisites](#prerequisites)
4. [Build and push the image](#build-and-push-the-image)
5. [Deploy on Minikube](#deploy-on-minikube)
6. [Using the application](#using-the-application)
7. [How DNS resolution works inside the cluster](#how-dns-resolution-works-inside-the-cluster)
8. [Resource requests and limits](#resource-requests-and-limits)
9. [Design choices and alternatives](#design-choices-and-alternatives)
10. [Testing scenarios and results](#testing-scenarios-and-results)
11. [Issues encountered](#issues-encountered)
12. [Known limitations](#known-limitations)
13. [Cleanup](#cleanup)
14. [Why use a Python virtual environment](#why-use-a-python-virtual-environment)

## Repository structure

```
.
├── app/
│   ├── app.py                   Flask application (MongoDB authentication added)
│   ├── requirements.txt         Pinned dependencies from the assignment
│   ├── Dockerfile
│   └── .dockerignore
├── k8s/
│   ├── mongo-secret.yaml        MongoDB credentials
│   ├── mongo-pv-pvc.yaml        PersistentVolume and PersistentVolumeClaim
│   ├── mongo-service.yaml       Headless Service (cluster-internal only)
│   ├── mongo-statefulset.yaml   MongoDB StatefulSet with authentication
│   ├── flask-deployment.yaml    Flask Deployment, 2 replicas
│   ├── flask-service.yaml       NodePort Service for Flask
│   └── flask-hpa.yaml           HorizontalPodAutoscaler
├── docs/                        Evidence from the autoscaling test
└── README.md
```

## Requirements checklist

| Requirement | Where it is met |
|---|---|
| Flask endpoints `/` and `/data` (GET and POST) | `app/app.py` |
| Flask code updated to authenticate with MongoDB | `app/app.py`, credentials read from environment variables |
| Flask Deployment with at least 2 replicas | `k8s/flask-deployment.yaml` (`replicas: 2`) |
| MongoDB StatefulSet with authentication enabled | `k8s/mongo-statefulset.yaml` |
| Flask Service reachable from the local machine | `k8s/flask-service.yaml` (NodePort) |
| MongoDB Service reachable only inside the cluster | `k8s/mongo-service.yaml` (ClusterIP, headless) |
| PV and PVC for MongoDB data | `k8s/mongo-pv-pvc.yaml`, mounted at `/data/db` |
| HPA at 70% CPU, minimum 2, maximum 5 | `k8s/flask-hpa.yaml` |
| DNS resolution explanation | [DNS section](#how-dns-resolution-works-inside-the-cluster) |
| Requests and limits (0.2 CPU and 250M, 0.5 CPU and 500M) on both workloads | Both workload manifests, explained [below](#resource-requests-and-limits) |
| Dockerfile and build and push instructions | `app/Dockerfile`, [build section](#build-and-push-the-image) |
| Design choices | [Design choices](#design-choices-and-alternatives) |
| Testing of autoscaling and database interactions | [Testing](#testing-scenarios-and-results) |

## Prerequisites

* Docker Desktop (or another Docker engine)
* `kubectl`
* Minikube
* Optional, only to run the application outside Kubernetes: Python 3.8 or later

## Build and push the image

The image is already published, so this step is only needed to use a different registry or to change the code.

```bash
docker login

# Multi-architecture build and push (works on Intel/AMD and Apple Silicon clusters)
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <your-dockerhub-user>/flask-mongo:v1 --push ./app

# Optional: confirm both architectures were pushed
docker buildx imagetools inspect <your-dockerhub-user>/flask-mongo:v1
```

If you use your own image, change the `image:` line in `k8s/flask-deployment.yaml`. Make sure the repository is public, or add an image pull secret to the Deployment.

If the build fails with "Multi-platform build is not supported for the docker driver", run `docker buildx create --name multi --use` once and repeat the command.

## Deploy on Minikube

```bash
# 1. Start the cluster and enable resource metrics (required by the HPA)
minikube start --driver=docker --cpus=2 --memory=4096
minikube addons enable metrics-server

# 2. Deploy MongoDB: Secret, storage, Service, StatefulSet
kubectl apply -f k8s/mongo-secret.yaml
kubectl apply -f k8s/mongo-pv-pvc.yaml
kubectl apply -f k8s/mongo-service.yaml
kubectl apply -f k8s/mongo-statefulset.yaml
kubectl rollout status statefulset/mongo

# 3. Deploy the Flask application: Deployment, Service, HPA
kubectl apply -f k8s/flask-deployment.yaml
kubectl apply -f k8s/flask-service.yaml
kubectl apply -f k8s/flask-hpa.yaml
kubectl rollout status deployment/flask-app

# 4. Check the result
kubectl get pods,svc,pv,pvc,hpa
```

The order matters. The Secret must exist before either workload starts, because both read their credentials from it. After step 4, expect `mongo-0` and two `flask-app-...` pods in `1/1 Running`, the PVC `Bound`, and the HPA showing a CPU percentage (it may show `<unknown>` for the first minute while metrics load).

## Using the application

On macOS with the Docker driver, a NodePort is not directly reachable from the host. Run this and keep the terminal open for as long as you use the application:

```bash
minikube service flask-service --url
```

It prints a URL such as `http://127.0.0.1:50405`. In a second terminal:

```bash
URL=http://127.0.0.1:50405    # use the URL printed above

curl $URL/
curl -X POST -H "Content-Type: application/json" -d '{"city":"Noida"}' $URL/data
curl $URL/data
```

Expected responses: the welcome message with the current time, then `{"status":"Data inserted"}`, then `[{"city":"Noida"}]`.

On Linux, `minikube ip` together with the NodePort `30080` also works. As an alternative on any platform, `kubectl port-forward svc/flask-service 5000:5000` exposes the application on `http://localhost:5000`.

## How DNS resolution works inside the cluster

Kubernetes runs a DNS server (CoreDNS) as a Service in the `kube-system` namespace. The kubelet configures every pod so that `/etc/resolv.conf` points at this DNS server and includes search domains for the pod namespace:

```
<namespace>.svc.cluster.local   svc.cluster.local   cluster.local
```

For every Service, the cluster creates a DNS record named `<service>.<namespace>.svc.cluster.local`. Pods in the same namespace can use the short name, because the search domains complete it automatically.

How this applies here:

1. The Flask pods are configured with `MONGO_HOST=mongo`, the name of the MongoDB Service. The application never uses a pod IP, because pod IPs change whenever a pod is recreated.
2. When the Flask pod connects to `mongo`, the resolver expands it to `mongo.default.svc.cluster.local` and asks CoreDNS.
3. The MongoDB Service is headless (`clusterIP: None`), so CoreDNS answers with the IP address of the ready MongoDB pod directly, instead of a virtual Service IP. Because MongoDB runs as a StatefulSet, the pod also has a stable per-pod name, `mongo-0.mongo.default.svc.cluster.local`.
4. The connection then goes pod to pod over the cluster network.

Configuring the connection from Flask to MongoDB therefore needs only three things: the Service name as the host, the port 27017, and the credentials injected from the Secret. These are set as environment variables in `k8s/flask-deployment.yaml`, and `app/app.py` builds the connection URI from them.

Verification from this deployment:

```
$ kubectl exec deploy/flask-app -- python -c "import socket; print(socket.gethostbyname('mongo'))"
10.244.0.4

$ kubectl get pod mongo-0 -o wide        # IP column
10.244.0.4
```

The name `mongo` resolved to exactly the IP address of the `mongo-0` pod. The same mechanism was used by the load-generator pods in the autoscaling test, which called `http://flask-service:5000/` by name.

## Resource requests and limits

A request is the amount of CPU and memory a container is guaranteed. The scheduler uses requests to decide which node has enough capacity for a pod. A limit is the maximum a container may use. The two behave differently when exceeded:

* CPU is compressible. A container that tries to use more CPU than its limit is throttled, but it keeps running.
* Memory is not compressible. A container that exceeds its memory limit is terminated by the kernel (OOMKilled) and restarted.

Requests protect a workload from starvation, and limits protect the node and neighbouring pods from a runaway workload. Because the requests are lower than the limits, both pods are in the Burstable quality-of-service class: they can use spare capacity, but they are not guaranteed it.

Both the Flask pods and the MongoDB pod use the values from the assignment:

| | CPU | Memory |
|---|---|---|
| Request | `200m` (0.2 CPU) | `250M` |
| Limit | `500m` (0.5 CPU) | `500M` |

Two related points:

* The HPA measures utilization as a percentage of the CPU request. With a request of 200m, the 70% target means about 140m of CPU per pod. Without a CPU request, the HPA could not calculate utilization at all.
* MongoDB sizes its internal cache from the memory of the node, not of the container. The StatefulSet therefore passes `--wiredTigerCacheSizeGB 0.25`, so the cache stays inside the 500M container limit.

Confirmed on the running pods with `kubectl describe pod`:

```
Limits:   cpu: 500m   memory: 500M
Requests: cpu: 200m   memory: 250M
```

## Design choices and alternatives

**Deployment for Flask, StatefulSet for MongoDB.** The Flask pods are stateless and interchangeable, which is what a Deployment is for. MongoDB needs a stable identity and stable storage, which is what a StatefulSet provides. A Deployment with a PVC would also work for one replica, but pods have no stable names and a rolling update could briefly run two pods against the same volume. The assignment also asks for a StatefulSet.

**One MongoDB replica, no replica set.** This is a single-node local cluster and a single PersistentVolume, so a three-member replica set would add complexity without adding real resilience. The trade-off is that the database is a single point of failure. In a production cluster I would use a replica set with `volumeClaimTemplates` so that each member has its own volume.

**Headless ClusterIP Service for MongoDB.** The database must be reachable only from inside the cluster, so the Service type is ClusterIP. Making it headless matches the StatefulSet `serviceName` and gives stable per-pod DNS names. NodePort and LoadBalancer were rejected because they would expose the database outside the cluster.

**NodePort Service for Flask.** It is the simplest way to reach the application from the host on Minikube and needs no extra components. A LoadBalancer Service needs `minikube tunnel`, and an Ingress needs an additional controller. Both are reasonable for a real environment but are more than this assignment requires. On macOS with the Docker driver, `minikube service --url` is still needed to reach the NodePort.

**Explicit PV and PVC with `hostPath`.** The assignment asks for a PV and a PVC, and Minikube stores `hostPath` data on the node. The PVC sets `storageClassName: manual` and `volumeName: mongo-pv`, which stops Minikube's default dynamic provisioner from creating a different volume and makes the PVC bind to the intended PV. The reclaim policy is `Retain` so that the data is not deleted if the claim is removed. On a cloud cluster I would use dynamic provisioning through a StorageClass instead of `hostPath`.

**Authentication through the official image variables.** The MongoDB image creates the root user from `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` the first time it starts with an empty data directory, and then runs with authentication required. The values come from a Kubernetes Secret, so credentials do not appear in the Deployment, the StatefulSet, or the application code. One consequence: changing the password later has no effect unless the data directory is also reset.

**Secret handling.** Kubernetes Secrets are only base64-encoded, not encrypted, and this demo Secret is committed so that the project can be reproduced. In a real environment I would create the Secret outside version control or use Sealed Secrets or an external secret manager, and I would enable encryption at rest.

**Minimal change to the application.** `app/app.py` keeps the endpoints from the assignment. It adds MongoDB authentication through environment variables (`MONGO_HOST`, `MONGO_PORT`, `MONGO_USERNAME`, `MONGO_PASSWORD`, with `authSource=admin`), URL-encodes the credentials, sets a 5 second server selection timeout, and returns HTTP 400 when a POST has no JSON body. The Flask development server from the assignment was kept. For production I would run the application under a WSGI server such as gunicorn.

**Image.** `python:3.9-slim` is small and is compatible with the pinned dependencies (Flask 2.0.2, Werkzeug 2.0.3, pymongo 3.12.0). The `requirements.txt` file is copied and installed before the code, so that code changes do not invalidate the cached dependency layer. The container runs as a non-root user. The image is tagged `v1` instead of `latest`, which keeps deployments reproducible and rollbacks possible. It was built for both `amd64` and `arm64`, because it was built on an Apple Silicon machine and a reviewer may use an Intel or AMD machine.

**Probes.** Readiness and liveness probes call `/`, which does not touch MongoDB. A database outage therefore makes the application return errors on `/data` but does not cause Flask pods to be restarted in a loop.

**HPA.** The `autoscaling/v2` HPA targets average CPU utilization of 70% of the request, with a minimum of 2 and a maximum of 5 replicas. The Deployment also starts with 2 replicas, which matches the HPA minimum.

## Testing scenarios and results

All results below come from this deployment. Times in the autoscaling test are host clock times (IST, UTC+05:30) on 19 November 2026.

### 1. MongoDB authentication

Without credentials, access is refused. With credentials, it succeeds.

```
$ kubectl exec mongo-0 -- mongosh --quiet --eval "db.adminCommand('listDatabases')"
MongoServerError: command listDatabases requires authentication

$ kubectl exec mongo-0 -- mongosh --quiet -u admin -p S3cureDemoPass \
    --authenticationDatabase admin --eval "db.adminCommand('listDatabases')"
{ databases: [ { name: 'admin', ... }, { name: 'config', ... }, { name: 'local', ... } ], ok: 1 }
```

In the local test before containerizing, starting the application with a wrong password made `GET /data` return HTTP 500, with `pymongo.errors.OperationFailure: Authentication failed.` (code 18, `AuthenticationFailed`). This shows that the application depends on valid credentials.

### 2. Application and database interaction through the cluster

```
$ curl $URL/
Welcome to the Flask app! The current time is: 2026-11-18 22:40:01.047647
$ curl -X POST -H "Content-Type: application/json" -d '{"city":"Noida"}' $URL/data
{"status":"Data inserted"}
$ curl $URL/data
[{"city":"Noida"}]
```

The record was then read directly from the database pod:

```
$ kubectl exec mongo-0 -- mongosh --quiet -u admin -p S3cureDemoPass --authenticationDatabase admin \
    --eval "db.getSiblingDB('flask_db').data.find({}, {_id:0}).toArray()"
[ { city: 'Noida' } ]
```

### 3. Data directory is on the persistent volume

`/data/db` inside the pod and `/data/mongo` on the Minikube node list the same files (`WiredTiger`, `mongod.lock`, `storage.bson`, collection and index files). The PVC `mongo-pvc` showed `Bound` to `mongo-pv` (1Gi, `RWO`).

### 4. Persistence across pod deletion

```
$ kubectl delete pod mongo-0
```

The StatefulSet recreated `mongo-0` immediately, and it was `1/1 Running` about 10 seconds later. The stored record was still present afterwards, both when queried directly in MongoDB and through the application (`[{"city":"Noida"}]`). The Flask pods were not restarted.

### 5. Traffic is shared by both Flask replicas

The probes of both pods write to the logs every 10 seconds, so requests were tagged with a query string to tell them apart from probe traffic.

```
$ for i in $(seq 1 20); do curl -s "$URL/?src=lbtest" > /dev/null; done
$ kubectl logs <flask-pod-1> | grep -c lbtest      13
$ kubectl logs <flask-pod-2> | grep -c lbtest      7
```

The 20 requests were split 13 and 7. After a second batch of 20, the cumulative counts were 19 and 21. The split is not even because kube-proxy chooses a pod for each new connection, but both replicas serve traffic.

### 6. Autoscaling under load

**Method.** Load was generated from inside the cluster, so that the local tunnel would not be the bottleneck. A `kubectl top pods` and `kubectl get hpa` snapshot was recorded every 15 seconds.

```bash
# Generator 1: requests the root endpoint
kubectl run load-generator-1 --rm -i --tty --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://flask-service:5000/ > /dev/null; done"

# Generator 2 (added later): requests /data, which also reads from MongoDB
kubectl run load-generator-2 --rm -i --tty --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://flask-service:5000/data > /dev/null; done"
```

**Results: scale-up.**

| Time | HPA target (CPU) | Replicas | Observation |
|---|---|---|---|
| 04:26:03 | 1% / 70% | 2 | Baseline, each Flask pod using about 2m |
| 04:27:04 | 1% / 70% | 2 | Generator 1 started, Flask pods at 85m and 79m |
| 04:28:19 | 79% / 70% | 2 | Pods at 161m and 157m, above target |
| 04:28:35 | 79% / 70% | 3 | First scale-up, about 1.5 minutes after the load started |
| 04:30 to 04:35 | 54% to 65% / 70% | 3 | Load shared by three pods, below target |
| 04:35:07 | 65% / 70% | 3 | Generator 2 started |
| 04:36:08 | 109% / 70% | 3 | Pods at about 218m each, MongoDB CPU up from 3m to 58m |
| 04:36:23 | 109% / 70% | 5 | Second scale-up, to the configured maximum |
| 04:37:08 | 84% / 70% | 5 | Five pods serving traffic |
| 04:38 to 04:45 | 68% to 70% / 70% | 5 | Stable at the maximum |

Both scale-ups match the HPA formula, `desired = ceil(current replicas x current utilization / target)`:

* 2 replicas at 79%: ceil(2 x 79 / 70) = ceil(2.26) = 3
* 3 replicas at 109%: ceil(3 x 109 / 70) = ceil(4.67) = 5

At 04:37:08 the utilization was 84% with 5 replicas, which would call for ceil(5 x 84 / 70) = 6, but the replica count stayed at 5. This confirms that `maxReplicas: 5` is enforced. The HPA events recorded by Kubernetes agree (`docs/hpa-events-scaleup.txt`):

```
Normal  SuccessfulRescale  New size: 3; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  New size: 5; reason: cpu resource utilization (percentage of request) above target
```

New pods took about 10 to 12 seconds to go from creation to `1/1 Running`, and no pod stayed in `Pending`.

**Results: scale-down.** The load-generator pods were deleted at about 04:51 (they appear in the 04:51:01 reading and are gone at 04:51:16).

| Time | HPA target (CPU) | Replicas | Observation |
|---|---|---|---|
| 04:52:17 | 10% / 70% | 5 | CPU has dropped, replicas unchanged |
| 04:53:17 | 1% / 70% | 5 | Flask pods at 2m each |
| 04:58:03 | 1% / 70% | 5 | Still 5 replicas |
| 04:58:19 | 1% / 70% | 2 | Scaled back to the minimum |

The replica count fell from 5 directly to 2, about 7 minutes after the load stopped. This is consistent with the HPA default scale-down stabilization window of 5 minutes, during which the highest recent recommendation is kept. Three pods were terminated and the two oldest remained. The log of this period is in `docs/autoscale-log-scaledown.txt`, and the events are in `docs/hpa-events-final.txt`.

## Issues encountered

* **Tunnel on macOS.** On macOS with the Docker driver, `minikube service flask-service --url` must keep running. When its terminal was closed, requests to the old port failed with `curl: (7) Failed to connect ... Couldn't connect to server`. Running the command again printed a new port (50405 instead of 50304) and the application responded normally. This was a tunnel problem, not an application problem, and the data was intact.
* **The first load generator was not enough.** The `/` endpoint does very little work. One generator raised average CPU to 79% of the request, which was enough for one scale-up (2 to 3) but then stabilized below the target. A second generator that requests `/data` (which also queries MongoDB) was needed to reach 109% and the maximum of 5 replicas.
* **Load generator connection errors.** The busybox generators printed many `wget: can't connect to remote host ... Cannot assign requested address` errors. busybox `wget` opens a new connection for every request, and a tight loop most likely exhausted the source ports of the generator pod, so many generated requests failed. The generators also used about 300m to 680m CPU on the same 2-CPU node as the application. The results therefore show how the HPA behaves, not the maximum capacity of the application.
* **Overwritten log file.** The first autoscaling log file was overwritten when logging was restarted without appending, so the raw scale-up readings were not kept in `docs/`. The scale-up table above was copied from the terminal output, and the scale-up is independently confirmed by the Kubernetes events in `docs/hpa-events-scaleup.txt`.

## Known limitations

* MongoDB runs as a single instance on a `hostPath` volume, so it is a single point of failure and the data lives on one node.
* The demo credentials are stored in a committed Secret and are only base64-encoded.
* The Flask development server is used, as in the provided code.
* Resource consumption of the load generators and the application share one small node, as described above.

## Cleanup

```bash
kubectl delete -f k8s/          # removes all resources (the PV is Retain, so node data remains)
minikube delete                 # removes the cluster and the node data
```

## Why use a Python virtual environment

A virtual environment (`python3 -m venv venv`) gives each project its own isolated set of installed packages.

* **Dependency isolation.** This project pins Flask 2.0.2, Werkzeug 2.0.3 and pymongo 3.12.0. Another project on the same machine may need newer versions. In a shared environment the two would conflict, but each virtual environment keeps its own versions.
* **Reproducibility.** The environment can be recreated from `requirements.txt` on any machine, which is the same install that the Docker image performs. What works locally therefore matches what runs in the container.
* **A clean system Python.** Installing packages globally can break operating system tools that rely on the system Python, and it may need administrator rights. A virtual environment avoids both.
* **Easy to discard.** If the environment breaks, deleting the `venv` folder and recreating it is cheap and does not affect anything else.
* **Clear dependency tracking.** Only the packages the project actually uses are installed, so `pip freeze` and `requirements.txt` reflect real dependencies.

The `venv` folder is excluded from Git (`.gitignore`) and from the Docker build context (`.dockerignore`).

To run the application locally without Kubernetes (Part 1 of the assignment):

```bash
cd app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
docker run -d -p 27017:27017 --name mongodb-local \
  -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=secret mongo:6
export MONGO_HOST=localhost MONGO_USERNAME=admin MONGO_PASSWORD=secret
python app.py
```