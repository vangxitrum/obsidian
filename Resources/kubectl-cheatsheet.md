---
type: reference
title: kubectl Cheat Sheet
tags: [kubernetes, kubectl, devops, cheatsheet]
created: 2026-07-06
---

# kubectl Cheat Sheet

Quick reference for the most useful `kubectl` commands, grouped by task.
`-n <ns>` scopes to a namespace; `-A`/`--all-namespaces` spans all. `-o wide` adds columns.

## Cluster info

```bash
kubectl version --short                 # client + server version
kubectl cluster-info                    # control plane + services endpoints
kubectl api-resources                   # all resource types (+ short names)
kubectl api-versions                    # supported API group versions
kubectl get componentstatuses          # control-plane component health
kubectl config current-context          # active context
```

## Get / list resources

```bash
kubectl get pods                        # pods in current namespace
kubectl get pods -A -o wide             # all namespaces, extra columns (node, IP)
kubectl get pods --show-labels          # include labels column
kubectl get pods -l app=nginx           # filter by label selector
kubectl get pods --field-selector status.phase=Running
kubectl get all                         # pods, svc, deploy, rs in namespace
kubectl get deploy,svc,ingress          # multiple kinds at once
kubectl get pods -w                     # watch for changes (stream)
kubectl get events --sort-by=.lastTimestamp
kubectl get pods --sort-by=.status.startTime
```

## Describe / inspect

```bash
kubectl describe pod <pod>              # events, conditions, container state
kubectl describe node <node>            # capacity, allocated, taints
kubectl get pod <pod> -o yaml           # full manifest as stored
kubectl explain deployment.spec.template  # schema docs for a field
```

## Logs

```bash
kubectl logs <pod>                      # stdout of single-container pod
kubectl logs <pod> -c <container>       # specific container
kubectl logs -f <pod>                   # follow (stream)
kubectl logs --previous <pod>           # last crashed container's logs
kubectl logs -l app=nginx --tail=100    # across pods matching label
kubectl logs --since=1h <pod>           # time-bounded
```

## Exec / attach / cp

```bash
kubectl exec -it <pod> -- /bin/sh       # interactive shell
kubectl exec <pod> -- env               # one-off command
kubectl attach -it <pod>                # attach to running process
kubectl cp <pod>:/path/file ./file      # copy out of container
kubectl cp ./file <pod>:/path/file      # copy into container
```

## Apply / create / delete

```bash
kubectl apply -f manifest.yaml          # declarative create/update
kubectl apply -f ./dir/ -R              # recursive over a directory
kubectl apply -k ./overlay/             # kustomize
kubectl create deployment web --image=nginx --replicas=3
kubectl create configmap cfg --from-file=./conf/
kubectl create secret generic db --from-literal=pass=secret
kubectl delete -f manifest.yaml
kubectl delete pod <pod> --grace-period=0 --force   # force kill
kubectl delete pods -l app=nginx        # by selector
kubectl apply -f - <<EOF                # inline manifest via heredoc
...
EOF
```

## Edit / patch

```bash
kubectl edit deployment <name>          # open live object in $EDITOR
kubectl set image deploy/web nginx=nginx:1.27   # update one container image
kubectl patch deploy web -p '{"spec":{"replicas":5}}'         # strategic merge
kubectl patch deploy web --type=json -p '[{"op":"replace","path":"/spec/replicas","value":5}]'
kubectl replace -f manifest.yaml        # full replace (must exist)
```

## Scale / rollout

```bash
kubectl scale deploy/web --replicas=5
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=80
kubectl rollout status deploy/web       # wait for rollout to finish
kubectl rollout history deploy/web      # revisions
kubectl rollout undo deploy/web         # roll back to previous
kubectl rollout undo deploy/web --to-revision=3
kubectl rollout restart deploy/web      # restart all pods (re-pull, reload)
kubectl rollout pause deploy/web        # freeze rollouts
kubectl rollout resume deploy/web
```

## Port-forward / proxy

```bash
kubectl port-forward pod/<pod> 8080:80          # local:remote
kubectl port-forward svc/<svc> 8080:80          # forward a service
kubectl port-forward deploy/web 8080:80
kubectl proxy --port=8001                        # API proxy on localhost
```

## Config / context

```bash
kubectl config get-contexts             # list contexts
kubectl config use-context <ctx>        # switch cluster/context
kubectl config set-context --current --namespace=<ns>   # pin default ns
kubectl config view --minify            # active context config only
```

## Namespaces

```bash
kubectl get ns
kubectl create namespace <ns>
kubectl delete namespace <ns>
kubectl get pods -n <ns>
kubectl -n <ns> get all
```

## Labels / annotations

```bash
kubectl label pod <pod> env=prod                 # add/overwrite label
kubectl label pod <pod> env-                     # remove label (trailing -)
kubectl annotate pod <pod> owner=team-a
kubectl get pods -L app,tier                     # show label values as columns
kubectl get pods -l 'env in (prod,staging)'      # set-based selector
```

## Node ops / drain

```bash
kubectl get nodes -o wide
kubectl cordon <node>                   # mark unschedulable
kubectl uncordon <node>                 # re-enable scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # evict for maintenance
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key:NoSchedule-        # remove taint (trailing -)
```

## Debug / troubleshoot

```bash
kubectl get pods --field-selector status.phase!=Running   # find unhealthy
kubectl describe pod <pod>              # check Events at bottom first
kubectl debug <pod> -it --image=busybox --target=<container>   # ephemeral debug container
kubectl debug node/<node> -it --image=busybox                  # node shell
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh # throwaway pod
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl top pod <pod> --containers      # per-container usage (needs metrics-server)
```

## Resource usage (top)

```bash
kubectl top nodes                       # node CPU/memory (needs metrics-server)
kubectl top pods -A                     # pod usage across namespaces
kubectl top pods --sort-by=memory
kubectl top pods --containers
```

## Wait

```bash
kubectl wait --for=condition=Ready pod/<pod> --timeout=120s
kubectl wait --for=condition=Available deploy/web --timeout=300s
kubectl wait --for=delete pod/<pod> --timeout=60s
kubectl wait --for=jsonpath='{.status.phase}'=Running pod/<pod>
```

## Output / jsonpath formatting

```bash
kubectl get pods -o json
kubectl get pods -o name                                     # just resource/names
kubectl get pod <pod> -o jsonpath='{.status.podIP}'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\\n"}}{{end}}'
kubectl get svc <svc> -o jsonpath='{.spec.clusterIP}'
```

## Handy flags (any command)

```bash
--dry-run=client -o yaml    # generate a manifest without applying (great for scaffolding)
-o wide                     # extra columns
-w / --watch                # stream changes
-v=6..9                     # verbose request/response logging
--as=<user> --as-group=<g>  # impersonate (RBAC testing)
```
