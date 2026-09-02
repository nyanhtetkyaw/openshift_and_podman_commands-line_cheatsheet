# OpenShift & Podman Command-Line Cheat Sheet

> This is the CLI reference for `oc`/`podman` interactive/day-to-day use.

---

## Table of Contents

**Podman**
[1. Images](#1-images) · [2. Containers — Lifecycle](#2-containers--lifecycle) · [3. Containers — Inspecting & Interacting](#3-containers--inspecting--interacting) · [4. Networking](#4-networking) · [5. Volumes](#5-volumes) · [6. Building Images](#6-building-images-containerfile) · [7. Pods](#7-pods) · [8. systemd Integration (Quadlet)](#8-systemd-integration-quadlet) · [9. Registry & Auth](#9-registry--auth) · [10. Cleanup](#10-cleanup)

**OpenShift**
[11. Login & Context](#11-login--context) · [12. Projects/Namespaces](#12-projectsnamespaces) · [13. Generic Resource Commands](#13-generic-resource-commands) · [14. Deployments & Rollouts](#14-deployments--rollouts) · [15. Builds & `oc new-app`](#15-builds--oc-new-app) · [16. Routes & Services](#16-routes--services) · [17. Scaling & Autoscaling](#17-scaling--autoscaling) · [18. ConfigMaps & Secrets](#18-configmaps--secrets) · [19. Logs, Exec & Debugging](#19-logs-exec--debugging) · [20. RBAC (`oc adm policy`)](#20-rbac-oc-adm-policy) · [21. Nodes & Cluster Health](#21-nodes--cluster-health) · [22. Resource Quotas & Limits](#22-resource-quotas--limits)

---

# PODMAN

## 1. Images

| Command                         | Example                                                      | Notes                               |
| ------------------------------- | ------------------------------------------------------------ | ----------------------------------- |
| `podman pull <image>`           | `podman pull registry.access.redhat.com/ubi9/ubi-minimal:latest` | Pulls without running               |
| `podman images`                 | `podman images`                                              | List locally stored images          |
| `podman images --filter <expr>` | `podman images --filter dangling=true`                       | Filter the listing                  |
| `podman inspect <image>`        | `podman inspect ubi9/ubi-minimal:latest`                     | Full JSON metadata                  |
| `podman image tree <image>`     | `podman image tree myapp:1.0`                                | Show the image's layer tree         |
| `podman tag <src> <dst>`        | `podman tag myapp:1.0 registry.example.com/myapp:1.0`        | Add a second tag (e.g. before push) |
| `podman push <image>`           | `podman push registry.example.com/myapp:1.0`                 | Push to a registry                  |
| `podman rmi <image>`            | `podman rmi myapp:1.0`                                       | Remove an image                     |
| `podman save -o <file> <image>` | `podman save -o myapp.tar myapp:1.0`                         | Export to a tarball                 |
| `podman load -i <file>`         | `podman load -i myapp.tar`                                   | Import from a tarball               |
| `podman search <term>`          | `podman search --limit 5 ubi9`                               | Search configured registries        |

---

## 2. Containers — Lifecycle

| Command                     | Example                                                      | Notes                                               |
| --------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| `podman run`                | `podman run -d --name web -p 8080:80 nginx:latest`           | `-d` detached, `-p host:container`                  |
| `podman run --rm`           | `podman run --rm -it ubi9/ubi-minimal:latest bash`           | Auto-remove on exit; `-it` interactive TTY          |
| `podman run` (env + volume) | `podman run -d --name app -e APP_ENV=prod -v ./data:/data:Z myapp:1.0` | `:Z` relabels for SELinux (see §5)                  |
| `podman start`              | `podman start web`                                           | Start a stopped container                           |
| `podman stop`               | `podman stop web`                                            | Graceful stop (SIGTERM, then SIGKILL after timeout) |
| `podman restart`            | `podman restart web`                                         |                                                     |
| `podman pause` / `unpause`  | `podman pause web`                                           | Freeze/unfreeze processes                           |
| `podman rm`                 | `podman rm web`                                              | Remove a stopped container                          |
| `podman rm -f`              | `podman rm -f web`                                           | Force-remove a running container                    |
| `podman ps`                 | `podman ps`                                                  | List running containers                             |
| `podman ps -a`              | `podman ps -a`                                               | Include stopped containers                          |
| `podman create`             | `podman create --name web nginx:latest`                      | Create without starting                             |

---

## 3. Containers — Inspecting & Interacting

| Command                  | Example                                                     | Notes                                         |
| ------------------------ | ----------------------------------------------------------- | --------------------------------------------- |
| `podman logs`            | `podman logs -f --tail 100 web`                             | `-f` follow, `--tail N` last N lines          |
| `podman exec`            | `podman exec -it web bash`                                  | Shell into a running container                |
| `podman inspect`         | `podman inspect web \| jq '.[0].NetworkSettings.IPAddress'` | Full container JSON                           |
| `podman top`             | `podman top web`                                            | Processes inside the container                |
| `podman stats`           | `podman stats --no-stream web`                              | CPU/mem/net usage; `--no-stream` for one shot |
| `podman port`            | `podman port web`                                           | Show published port mappings                  |
| `podman cp`              | `podman cp web:/app/log.txt ./log.txt`                      | Copy files in/out of a container              |
| `podman diff`            | `podman diff web`                                           | Filesystem changes since the image            |
| `podman commit`          | `podman commit web myapp:debug-snapshot`                    | Snapshot a running container to an image      |
| `podman healthcheck run` | `podman healthcheck run web`                                | Manually trigger a defined healthcheck        |

---

## 4. Networking

| Command                     | Example                                                 | Notes                                   |
| --------------------------- | ------------------------------------------------------- | --------------------------------------- |
| `podman network create`     | `podman network create app-net --subnet 10.89.1.0/24`   | Custom bridge network                   |
| `podman network ls`         | `podman network ls`                                     | List networks                           |
| `podman network inspect`    | `podman network inspect app-net`                        | Full network JSON                       |
| `podman network rm`         | `podman network rm app-net`                             | Remove a network                        |
| `podman run --network`      | `podman run -d --network app-net --name db postgres:16` | Attach at run time                      |
| `podman network connect`    | `podman network connect app-net web`                    | Attach a running container to a network |
| `podman network disconnect` | `podman network disconnect app-net web`                 | Detach                                  |

---

## 5. Volumes

| Command                        | Example                                                      | Notes                                     |
| ------------------------------ | ------------------------------------------------------------ | ----------------------------------------- |
| `podman volume create`         | `podman volume create app-data`                              | Named, Podman-managed volume              |
| `podman volume ls`             | `podman volume ls`                                           |                                           |
| `podman volume inspect`        | `podman volume inspect app-data`                             | Shows the host mount point                |
| `podman volume rm`             | `podman volume rm app-data`                                  |                                           |
| `podman run -v` (named volume) | `podman run -d -v app-data:/var/lib/postgresql/data postgres:16` |                                           |
| `podman run -v` (bind mount)   | `podman run -d -v /srv/app-config:/etc/app:ro,Z myapp:1.0`   | `:ro` read-only, `:Z` for SELinux relabel |
| `podman volume prune`          | `podman volume prune`                                        | Remove all unused volumes                 |

> **SELinux note:** on RHEL, bind-mounting a host directory into a container almost always needs `:Z` (private label) or `:z` (shared label) or the container process will get `Permission denied` even though the Unix permissions look fine.

---

## 6. Building Images (Containerfile)

| Command                    | Example                                                    | Notes                                            |
| -------------------------- | ---------------------------------------------------------- | ------------------------------------------------ |
| `podman build`             | `podman build -t myapp:1.0 .`                              | Build from `./Containerfile` (or `./Dockerfile`) |
| `podman build -f`          | `podman build -f custom.Containerfile -t myapp:1.0 .`      | Explicit build-file path                         |
| `podman build --build-arg` | `podman build --build-arg APP_VERSION=1.0 -t myapp:1.0 .`  | Pass build-time variables                        |
| `podman build --no-cache`  | `podman build --no-cache -t myapp:1.0 .`                   | Force a clean rebuild of every layer             |
| `podman build --platform`  | `podman build --platform linux/arm64 -t myapp:1.0-arm64 .` | Cross-arch build                                 |

**Minimal `Containerfile` example:**

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
RUN microdnf install -y python3.11 && microdnf clean all
COPY app/ /opt/app/
WORKDIR /opt/app
RUN useradd -r -u 1001 appuser
USER 1001
EXPOSE 8080
ENTRYPOINT ["python3.11", "server.py"]
```

* `USER 1001` (non-root) matters doubly for OpenShift — by default it runs containers as an arbitrary high UID assigned per-namespace, and images that hardcode `USER root` or rely on a fixed low UID will fail their SCC check (see §22-adjacent OpenShift security context constraints).

---

## 7. Pods

Podman "pods" share a network namespace between containers, mirroring a Kubernetes/OpenShift Pod — useful for local testing of multi-container workloads before deploying them for real.

| Command                     | Example                                            | Notes                                                    |
| --------------------------- | -------------------------------------------------- | -------------------------------------------------------- |
| `podman pod create`         | `podman pod create --name app-pod -p 8080:8080`    | Publishes the port once, for the whole pod               |
| `podman run --pod`          | `podman run -d --pod app-pod --name web myapp:1.0` | Adds a container into an existing pod                    |
| `podman pod ps`             | `podman pod ps`                                    | List pods                                                |
| `podman pod inspect`        | `podman pod inspect app-pod`                       |                                                          |
| `podman pod stop` / `start` | `podman pod stop app-pod`                          | Acts on every container in the pod                       |
| `podman pod rm`             | `podman pod rm -f app-pod`                         |                                                          |
| `podman generate kube`      | `podman generate kube app-pod > app-pod.yaml`      | Emit a Kubernetes-style YAML manifest from a running pod |
| `podman play kube`          | `podman play kube app-pod.yaml`                    | Run a Kubernetes-style YAML manifest locally with Podman |

---

## 8. systemd Integration (Quadlet)

The modern (Podman 4.4+) way to run a container as a systemd service is a **Quadlet** unit file, not a hand-generated `.service` file.

`~/.config/containers/systemd/myapp.container`:

```ini
[Unit]
Description=My App container

[Container]
Image=myapp:1.0
PublishPort=8080:8080
Volume=app-data:/var/lib/myapp/data:Z
Environment=APP_ENV=production

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
$ systemctl --user daemon-reload
$ systemctl --user enable --now myapp.service
$ systemctl --user status myapp.service
```

* Rootless (`--user`) is the recommended default; drop `--user` and put the unit in `/etc/containers/systemd/` for a root-managed service instead.
* Older `podman generate systemd --new --name web > web.service` still works but is deprecated in favor of Quadlet as of Podman 4.4+.

---

## 9. Registry & Auth

| Command                 | Example                                                      | Notes                                             |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| `podman login`          | `podman login registry.example.com`                          | Prompts for credentials                           |
| `podman login` (inline) | `podman login -u deployer -p "$REG_TOKEN" registry.example.com` | For CI use                                        |
| `podman logout`         | `podman logout registry.example.com`                         |                                                   |
| `podman info`           | `podman info --format json \| jq '.registries'`              | Shows configured registries, storage driver, etc. |

`/etc/containers/registries.conf` (system-wide registry search order):

```toml
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]
```

---

## 10. Cleanup

| Command                  | Example                            | Notes                                               |
| ------------------------ | ---------------------------------- | --------------------------------------------------- |
| `podman container prune` | `podman container prune`           | Remove all stopped containers                       |
| `podman image prune`     | `podman image prune -a`            | `-a` also removes unused (not just dangling) images |
| `podman volume prune`    | `podman volume prune`              | Remove unused volumes                               |
| `podman network prune`   | `podman network prune`             | Remove unused networks                              |
| `podman system prune`    | `podman system prune -a --volumes` | Everything at once — use with care                  |
| `podman system df`       | `podman system df`                 | Disk usage breakdown by images/containers/volumes   |

---

# OPENSHIFT

## 11. Login & Context

| Command                   | Example                                                      | Notes                                |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| `oc login`                | `oc login https://api.cluster.example.com:6443 -u devuser`   | Prompts for password                 |
| `oc login` (token)        | `oc login --token=sha256~xxxx --server=https://api.cluster.example.com:6443` | Common for CI/scripted use           |
| `oc whoami`               | `oc whoami`                                                  | Current authenticated user           |
| `oc whoami --show-server` | `oc whoami --show-server`                                    | Current API endpoint                 |
| `oc whoami -t`            | `oc whoami -t`                                               | Print the current session token      |
| `oc config view`          | `oc config view --minify`                                    | Show the active context's kubeconfig |
| `oc config use-context`   | `oc config use-context prod-cluster`                         | Switch between saved contexts        |
| `oc logout`               | `oc logout`                                                  | Invalidate the current session token |

---

## 12. Projects/Namespaces

| Command             | Example                                        | Notes                            |
| ------------------- | ---------------------------------------------- | -------------------------------- |
| `oc new-project`    | `oc new-project myapp --display-name="My App"` | Create + switch into it          |
| `oc project`        | `oc project myapp`                             | Switch active project            |
| `oc projects`       | `oc projects`                                  | List all projects you can see    |
| `oc get project`    | `oc get project myapp -o yaml`                 | Inspect one project              |
| `oc delete project` | `oc delete project myapp`                      | Deletes everything inside it too |

---

## 13. Generic Resource Commands

These four verbs (`get`, `describe`, `apply`/`create`, `delete`) work identically across almost every resource kind.

| Command               | Example                                 | Notes                                                 |
| --------------------- | --------------------------------------- | ----------------------------------------------------- |
| `oc get`              | `oc get pods -n myapp`                  | `-n` targets a namespace without switching context    |
| `oc get -o wide`      | `oc get pods -o wide`                   | Extra columns (node, IP)                              |
| `oc get -o yaml/json` | `oc get deployment myapp -o yaml`       | Full manifest                                         |
| `oc get -w`           | `oc get pods -w`                        | Watch for changes live                                |
| `oc get all`          | `oc get all -n myapp`                   | Pods, services, deployments, routes, etc. in one shot |
| `oc describe`         | `oc describe pod myapp-7f9c-abcde`      | Human-readable detail + recent Events                 |
| `oc apply -f`         | `oc apply -f deployment.yaml`           | Declarative — create or update to match the file      |
| `oc create -f`        | `oc create -f deployment.yaml`          | Imperative — fails if it already exists               |
| `oc replace -f`       | `oc replace -f deployment.yaml --force` | Full replace, deletes+recreates if needed             |
| `oc delete`           | `oc delete pod myapp-7f9c-abcde`        |                                                       |
| `oc delete -f`        | `oc delete -f deployment.yaml`          | Delete everything defined in a manifest               |
| `oc edit`             | `oc edit deployment myapp`              | Opens the live resource in `$EDITOR`                  |
| `oc explain`          | `oc explain deployment.spec.template`   | Inline API field documentation                        |
| `oc api-resources`    | `oc api-resources`                      | List every resource kind the cluster supports         |

---

## 14. Deployments & Rollouts

| Command                         | Example                                                      | Notes                                                        |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `oc rollout status`             | `oc rollout status deployment/myapp`                         | Wait for a rollout to finish                                 |
| `oc rollout history`            | `oc rollout history deployment/myapp`                        | List revisions                                               |
| `oc rollout undo`               | `oc rollout undo deployment/myapp`                           | Roll back to the previous revision                           |
| `oc rollout undo --to-revision` | `oc rollout undo deployment/myapp --to-revision=3`           | Roll back to a specific revision                             |
| `oc rollout restart`            | `oc rollout restart deployment/myapp`                        | Force a rolling restart (e.g. to pick up a ConfigMap change) |
| `oc rollout pause` / `resume`   | `oc rollout pause deployment/myapp`                          | Freeze updates mid-rollout for inspection                    |
| `oc set image`                  | `oc set image deployment/myapp myapp=registry.example.com/myapp:1.1` | Update just the image, triggers a rollout                    |
| `oc set env`                    | `oc set env deployment/myapp APP_ENV=staging`                | Update an env var, triggers a rollout                        |

---

## 15. Builds & `oc new-app`

| Command                      | Example                                                      | Notes                                      |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| `oc new-app` (from image)    | `oc new-app registry.example.com/myapp:1.0 --name=myapp`     | Fast path: creates Deployment + Service    |
| `oc new-app` (from git, S2I) | `oc new-app python:3.11~https://github.com/example/myapp.git --name=myapp` | Source-to-Image build from a git repo      |
| `oc new-build`               | `oc new-build --binary --name=myapp -l app=myapp`            | Build config without deploying             |
| `oc start-build`             | `oc start-build myapp --from-dir=. --follow`                 | Trigger a build, `--follow` streams logs   |
| `oc get bc`                  | `oc get buildconfig`                                         | List BuildConfigs                          |
| `oc get builds`              | `oc get builds`                                              | List individual Build runs                 |
| `oc logs -f bc/`             | `oc logs -f bc/myapp`                                        | Stream the latest build's logs             |
| `oc cancel-build`            | `oc cancel-build myapp-3`                                    | Abort an in-progress build                 |
| `oc import-image`            | `oc import-image myapp:latest --from=registry.example.com/myapp:latest --confirm` | Track an external image via an ImageStream |

---

## 16. Routes & Services

| Command                               | Example                                                      | Notes                        |
| ------------------------------------- | ------------------------------------------------------------ | ---------------------------- |
| `oc expose` (service from deployment) | `oc expose deployment/myapp --port=8080`                     | Creates a Service            |
| `oc expose` (route from service)      | `oc expose service/myapp`                                    | Creates an external Route    |
| `oc expose --hostname`                | `oc expose service/myapp --hostname=app.example.com`         | Custom hostname              |
| `oc get routes`                       | `oc get routes`                                              | List routes and their URLs   |
| `oc get svc`                          | `oc get svc`                                                 | List services                |
| `oc create route edge`                | `oc create route edge myapp-tls --service=myapp --hostname=app.example.com` | TLS-terminated at the router |
| `oc annotate route`                   | `oc annotate route myapp haproxy.router.openshift.io/timeout=60s` | Router-specific tuning       |

---

## 17. Scaling & Autoscaling

| Command           | Example                                                      | Notes                                                        |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `oc scale`        | `oc scale deployment/myapp --replicas=4`                     | Manual scale                                                 |
| `oc autoscale`    | `oc autoscale deployment/myapp --min=2 --max=10 --cpu-percent=70` | Creates a HorizontalPodAutoscaler                            |
| `oc get hpa`      | `oc get hpa`                                                 | List autoscalers and current status                          |
| `oc adm top pods` | `oc adm top pods -n myapp`                                   | Live CPU/memory usage, needed to sanity-check HPA thresholds |

---

## 18. ConfigMaps & Secrets

| Command                            | Example                                                      | Notes                                      |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| `oc create configmap`              | `oc create configmap app-config --from-file=app.conf`        | From a file                                |
| `oc create configmap` (literal)    | `oc create configmap app-config --from-literal=LOG_LEVEL=info` | From key=value pairs                       |
| `oc create secret generic`         | `oc create secret generic app-secret --from-literal=DB_PASSWORD=changeme` | Opaque secret                              |
| `oc create secret tls`             | `oc create secret tls myapp-tls --cert=tls.crt --key=tls.key` | TLS secret (pairs with §16's `edge` route) |
| `oc create secret docker-registry` | `oc create secret docker-registry pull-secret --docker-server=registry.example.com --docker-username=u --docker-password=p` | For pulling from a private registry        |
| `oc set volume`                    | `oc set volume deployment/myapp --add -t configmap --configmap-name=app-config -m /etc/app` | Mount a ConfigMap into pods                |
| `oc extract`                       | `oc extract secret/app-secret --to=./secrets/`               | Pull secret values back out to files       |

---

## 19. Logs, Exec & Debugging

| Command              | Example                                           | Notes                                               |
| -------------------- | ------------------------------------------------- | --------------------------------------------------- |
| `oc logs`            | `oc logs -f myapp-7f9c-abcde`                     | `-f` follow                                         |
| `oc logs --previous` | `oc logs myapp-7f9c-abcde --previous`             | Logs from the previous (crashed) container instance |
| `oc logs -c`         | `oc logs myapp-7f9c-abcde -c sidecar`             | Specific container in a multi-container pod         |
| `oc exec`            | `oc exec -it myapp-7f9c-abcde -- bash`            | Shell into a running pod                            |
| `oc rsh`             | `oc rsh myapp-7f9c-abcde`                         | OpenShift shorthand for exec+shell                  |
| `oc port-forward`    | `oc port-forward myapp-7f9c-abcde 8080:8080`      | Tunnel a local port to the pod                      |
| `oc debug`           | `oc debug deployment/myapp`                       | Launches a debug pod copying the deployment's spec  |
| `oc debug node/`     | `oc debug node/worker-1.example.com`              | Privileged shell on a node itself                   |
| `oc get events`      | `oc get events --sort-by=.lastTimestamp -n myapp` | Recent cluster events, chronological                |
| `oc adm must-gather` | `oc adm must-gather`                              | Collect a full diagnostic bundle for support cases  |

---

## 20. RBAC (`oc adm policy`)

These are CLI shortcuts around the exact `Role`/`RoleBinding`/`ClusterRole`/`ClusterRoleBinding` objects covered in the Ansible cheat sheet's Section 33 — useful for one-off grants without writing a manifest.

| Command                                  | Example                                                      | Notes                                                  |
| ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| `oc adm policy add-role-to-user`         | `oc adm policy add-role-to-user edit jdoe -n myapp`          | Namespaced role grant                                  |
| `oc adm policy remove-role-from-user`    | `oc adm policy remove-role-from-user edit jdoe -n myapp`     |                                                        |
| `oc adm policy add-cluster-role-to-user` | `oc adm policy add-cluster-role-to-user cluster-reader jdoe` | Cluster-wide grant                                     |
| `oc adm policy add-role-to-group`        | `oc adm policy add-role-to-group edit platform-ops -n myapp` | Grant to a group instead of one user                   |
| `oc adm policy add-scc-to-user`          | `oc adm policy add-scc-to-user anyuid -z myapp-sa -n myapp`  | Grant a ServiceAccount a Security Context Constraint   |
| `oc get rolebindings`                    | `oc get rolebindings -n myapp`                               | Audit what's currently bound                           |
| `oc auth can-i`                          | `oc auth can-i update deployments --as=jdoe -n myapp`        | Verify effective permissions                           |
| `oc auth can-i --list`                   | `oc auth can-i --list -n myapp`                              | List everything the current user can do in a namespace |

---

## 21. Nodes & Cluster Health

| Command                   | Example                                                      | Notes                                          |
| ------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| `oc get nodes`            | `oc get nodes -o wide`                                       | List nodes, roles, versions                    |
| `oc describe node`        | `oc describe node worker-1.example.com`                      | Capacity, conditions, running pods             |
| `oc adm top nodes`        | `oc adm top nodes`                                           | Live CPU/memory usage per node                 |
| `oc adm cordon`           | `oc adm cordon worker-1.example.com`                         | Mark unschedulable (before maintenance)        |
| `oc adm uncordon`         | `oc adm uncordon worker-1.example.com`                       | Mark schedulable again                         |
| `oc adm drain`            | `oc adm drain worker-1.example.com --ignore-daemonsets --delete-emptydir-data` | Evict all pods before maintenance/decommission |
| `oc get clusteroperators` | `oc get co`                                                  | Health of every core cluster operator          |
| `oc get clusterversion`   | `oc get clusterversion`                                      | Current OpenShift version and update status    |
| `oc adm upgrade`          | `oc adm upgrade`                                             | Show/trigger available cluster upgrades        |

---

## 22. Resource Quotas & Limits

| Command                             | Example                                                      | Notes                                                        |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `oc create quota`                   | `oc create quota myapp-quota --hard=cpu=4,memory=8Gi,pods=20` | Namespace-wide resource cap                                  |
| `oc get resourcequota`              | `oc get resourcequota -n myapp`                              | Current usage vs. quota                                      |
| `oc describe quota`                 | `oc describe resourcequota myapp-quota -n myapp`             | Per-resource breakdown                                       |
| `oc create limitrange` (file-based) | `oc apply -f limitrange.yaml`                                | Default/min/max per-pod or per-container limits — no direct imperative `create limitrange` subcommand exists, so this is the manifest route |
| `oc set resources`                  | `oc set resources deployment/myapp --limits=cpu=500m,memory=512Mi --requests=cpu=250m,memory=256Mi` | Set container resource requests/limits directly              |



### Reference Links

**Podman**

- [Podman official docs](https://docs.podman.io/en/latest/)
- [Podman command reference](https://docs.podman.io/en/latest/Commands.html)
- [Quadlet (systemd units) reference](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [containers.podman Ansible collection](https://docs.ansible.com/projects/ansible/latest/collections/containers/podman/index.html) — `podman_image`, `podman_container`, `podman_generate_systemd`, etc.

**OpenShift / Kubernetes**

- [OpenShift CLI (`oc`) reference](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc)
- [kubectl reference docs](https://kubernetes.io/docs/reference/kubectl/)
- [kubernetes.core Ansible collection](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/index.html) — `k8s`, `k8s_info`, `k8s_scale`, `k8s_drain`, `helm`, etc.
- [OpenShift RBAC (roles & bindings) guide](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/using-rbac)
- [Kubernetes RBAC authorization](https://kubernetes.io/docs/reference/access-control/rbac/)

**Compliance / OpenSCAP**

- [OpenSCAP project](https://www.open-scap.org/)
- [OpenShift Compliance Operator docs](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/compliance-operator)

**Authentication (OIDC/SAML)**

- [OpenShift identity providers (OIDC/SAML) config](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/index)
- [Kubernetes OIDC authentication](https://kubernetes.io/docs/reference/access-control/authentication/#openid-connect-tokens) 

Happy learning!
