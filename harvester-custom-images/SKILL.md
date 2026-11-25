---
name: harvester-custom-images
description: Build images and deploy them to a local running Harvester cluster.
metadata:
  author: Volker Theile
  email: vtheile@suse.com
---

# harvester-custom-images

## When to use

Use this whenever it is necessary to build a custom container image of the Harvester component we are currently working on, and deploy it to a local test cluster.

## Instructions

Do not run this skill this skill unless specifically asked to do so.

### 1. Check the target cluster first

The target cluster should already be running and reachable via `kubectl`. Check before building anything:

```
kubectl get nodes -o wide
```

- If the API (VIP, e.g. `192.168.0.131`) is unreachable ("no route to host"), **abort and ask the user** to start or fix the cluster. Don't build in the meantime.
- Nodes are mostly 1 or 3 node clusters, deployed via https://github.com/harvester/ipxe-examples/tree/main/vagrant-pxe-harvester.
- SSH: `rancher@192.168.0.3[0|1|2]` (`rancher` has `sudo`).
- If SSH fails with `REMOTE HOST IDENTIFICATION HAS CHANGED`, **do not remove the key on your own**. Ask the user first. Only after confirmation run `ssh-keygen -R <IP>`. Do not use `StrictHostKeyChecking=no` to work around it.

### 2. Refresh the version file

`scripts/.version_env` is generated and is often stale (it holds an old commit hash). Always regenerate it before building, otherwise the image tag is wrong:

```
make gen-version-env
grep -E 'COMMIT=|TAG=' scripts/.version_env   # TAG=<hash>-amd64
```

### 3. Build and package

```
make build
make package                      # image: rancher/harvester:<hash>-amd64
make package-harvester-webhook    # image: rancher/harvester-webhook:<hash>-amd64
```

- `make package-all` also builds the upgrade image, which is usually not needed.
- Target names can differ in other repos; check the `Makefile`.
- Builds can take a while; use a long timeout (e.g. 30 min).
- `make package*` prints `Built <repo>:<tag>` and also tags `master-head`. Use the `<hash>-amd64` tag, not `master-head`.

### 4. Export, copy and load on **every** node

Do this for each node (.30, .31, .32), not just one:

```
docker save -o /tmp/<COMPONENT>.img rancher/<COMPONENT>:<hash>-amd64
for i in 30 31 32; do
  scp -O /tmp/<COMPONENT>.img rancher@192.168.0.$i:/home/rancher && \
  ssh rancher@192.168.0.$i 'sudo docker --address /run/k3s/containerd/containerd.sock --namespace k8s.io load -i /home/rancher/<COMPONENT>.img'
done
```

Each node must print `Loaded image: docker.io/rancher/<COMPONENT>:<hash>-amd64`. If one node fails, fix that node. Don't continue with a partial rollout.

### 5. Roll out to the cluster

Loading the image alone changes nothing. The deployment must point to the new tag:

```
kubectl -n harvester-system set image deploy/harvester         apiserver=rancher/harvester:<hash>-amd64
kubectl -n harvester-system set image deploy/harvester-webhook harvester-webhook=rancher/harvester-webhook:<hash>-amd64
kubectl -n harvester-system rollout status deploy/<name> --timeout=300s
```

Container names: `apiserver` (deploy `harvester`), `harvester-webhook` (deploy `harvester-webhook`). Verify them first with:

```
kubectl -n harvester-system get deploy <name> -o jsonpath='{.spec.template.spec.containers[*].name}'
```

### 6. Verify that the image is really applied

```
kubectl -n harvester-system get deploy harvester harvester-webhook -o wide
kubectl -n harvester-system get pods -l app.kubernetes.io/name=harvester \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"  "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

Check that all pods use the new tag, all are Ready (one per node for 3-node clusters), and the `imageID` is identical across the pods of a deployment.

### 7. Fleet drift (important caveat)

Harvester is managed by Fleet (`ManagedChart` `harvester` in `fleet-local`, bundle `mcc-harvester`). It wants `rancher/harvester:master-head` and `rancher/harvester-webhook:master-head`. After a manual `set image` the bundle shows **Modified**:

```
kubectl -n fleet-local get bundle mcc-harvester -o jsonpath='{.status.summary}'
```

- `correctDrift` is normally not set, so Fleet does not revert the change immediately.
- A resync (chart update, Fleet restart) can reset the deployments to `master-head`. On the nodes, that tag may point to an **old** image, which would silently roll back the code.
- Report this risk to the user. Optional mitigation, only with the user's agreement: also load the image under the `master-head` tag on all nodes (`docker save` with both tags), so a Fleet resync keeps the new code.
- Don't pause or edit the ManagedChart without asking.

### 8. Summary for the user

At the end, report: commit hash and tags, which nodes got the image, rollout status, and the Fleet drift note. Mention any component you did not build (e.g. the webhook) and offer to do it.

