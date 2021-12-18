# doks-vcluster-challenge

Aroused by https://www.digitalocean.com/community/pages/kubernetes-challenge to give `vcluster` a shot.

## Install vcluster CLI (locally)
Once the DOKS is created and the `kubectl` CLI is configured properly, we are ready to go. First thing first, we have to download and install the `vcluster` CLI locally by following the [docs](https://www.vcluster.com/docs/getting-started/setup#download-vcluster-cli). 


## Deploy
All set. Let's create a new vcluster called `team-1` in namespace `ns-1` 
```bash
$ vcluster create team-1 -n ns-1
[fatal]  seems like helm is not installed. Helm is required for the creation of a virtual cluster. Please visit https://helm.sh/docs/intro/install/ for install instructions
```
Aha, the docs didn't mention this pre-requisite. But it's alright, we can manage it.
A few seconds later, `helm` CLI is installed 👌. Let's retry.

```bash
$ vcluster create team-1 -n ns-1
[info]   Creating namespace ns-1
[info]   execute command: helm upgrade team-1 vcluster --repo https://charts.loft.sh --version 0.4.5 --kubeconfig /tmp/921674663 --namespace ns-1 --install --repository-config='' --values /tmp/307639514
[done] √ Successfully created virtual cluster team-1 in namespace ns-1. Use 'vcluster connect team-1 --namespace ns-1' to access the virtual cluster
```

As it's literal meaning, vcluster seemingly uses helm under the hood to make all magics happen. Let's sneak in a bit what are applied from the chart.

```bash
$ helm -n ns-1 get manifest team-1 | grep Source
# Source: vcluster/templates/serviceaccount.yaml
# Source: vcluster/templates/rbac/role.yaml
# Source: vcluster/templates/rbac/rolebinding.yaml
# Source: vcluster/templates/service.yaml
# Source: vcluster/templates/statefulset-service.yaml
# Source: vcluster/templates/statefulset.yaml
```
Surprisingly, vcluster works with just a few manifests. No wonder it's advertised [No Admin Privileges Required](https://www.vcluster.com/docs/architecture/basics#5-no-admin-privileges-required):
>  If a user has the RBAC permissions to deploy a simple web application to a namespace, they should also be able to deploy vclusters to this namespace.

## Use the created vcluster
As suggested, running the connect command and start port-forwarding.
```bash
$ vcluster connect team-1 -n ns-1
[done] √ Virtual cluster kube config written to: ./kubeconfig.yaml. You can access the cluster via `kubectl --kubeconfig ./kubeconfig.yaml get namespaces`
[info]   Starting port-forwarding at 8443:8443
Forwarding from 127.0.0.1:8443 -> 8443
Forwarding from [::1]:8443 -> 844
```
So, from now on, we can run `kubectl --kubeconfig ./kubeconfig.yaml <sub-command>` against this virtual cluster without knowing whether it's real. Carrying the `--kubeconfig` maybe a bit lame but let's stick with it for now.

```
$ kubectl --kubeconfig ./kubeconfig.yaml get namespace
NAME              STATUS   AGE
default           Active   21h
kube-system       Active   21h
kube-public       Active   21h
kube-node-lease   Active   21h
```
It does look like a brand new cluster. 

## Deploy apps and test k8s Service
Running:

`kubectl --kubeconfig ./kubeconfig.yaml apply -f doks-vcluster-challenge/server.yaml`

and

```bash
$ kubectl --kubeconfig ./kubeconfig.yaml get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/server-5bc67676fd-9rc8r   1/1     Running   0          64s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.245.48.66    <none>        443/TCP   22h
service/server       ClusterIP   10.245.98.125   <none>        80/TCP    5m23s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/server   1/1     1            1           5m24s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/server-5bc67676fd   1         1         1       65s
```

Alright. But then, how's the real cluster looks like?

```bash
$ KUBECONFIG="" kubectl get -n ns-1 all
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/coredns-7448499f4d-tfn7w-x-kube-system-x-team-1   1/1     Running   0          22h
pod/server-5bc67676fd-9rc8r-x-default-x-team-1        1/1     Running   0          4m12s
pod/team-1-0                                          2/2     Running   0          22h

NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns-x-kube-system-x-team-1   ClusterIP   10.245.32.164    <none>        53/UDP,53/TCP,9153/TCP   22h
service/server-x-default-x-team-1         ClusterIP   10.245.98.125    <none>        80/TCP                   8m31s
service/team-1                            ClusterIP   10.245.48.66     <none>        443/TCP                  22h
service/team-1-headless                   ClusterIP   None             <none>        443/TCP                  22h
service/team-1-node-9nn58                 ClusterIP   10.245.208.244   <none>        10250/TCP                4m12s
service/team-1-node-s2qpb                 ClusterIP   10.245.17.19     <none>        10250/TCP                22h

NAME                      READY   AGE
statefulset.apps/team-1   1/1     22h
```

As we can see, both server and pod are appended `x-<inner-namespace>-x-<vcluster-name>` suffix. vcluster uses **syncer** to mimic the workloads are logically in a virtual cluster according to its documentation.

## Within vcluster traffic

Let's try to have a pod in the vcluster sends a request to another pod via the k8s service in the same vcluster.

```bash
kubectl --kubeconfig ./kubeconfig.yaml create job --image=curlimages/curl curl -- curl -v http://server/anything

$ kubectl --kubeconfig ./kubeconfig.yaml logs curl-c9nth
* Connected to server (10.245.98.125) port 80 (#0)
> GET /anything HTTP/1.1
> Host: server
> User-Agent: curl/7.80.0-DEV
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: gunicorn/19.9.0
< Date: Fri, 17 Dec 2021 11:53:21 GMT
< Connection: keep-alive
< Content-Type: application/json
< Content-Length: 267
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< 
{ [267 bytes data]
100   267  100   267    0     0  10086      0 --:--:-- --:--:-- --:--:-- 10269
* Connection #0 to host server left intact
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "server", 
    "User-Agent": "curl/7.80.0-DEV"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "10.244.1.228", 
  "url": "http://server/anything"
}
```
That is not a surprise. vcluster has it documented how it's achieved by having another DNS service in the vcluster instead of talking to the "outer" DNS service directly in the real cluster.

On the other hand, we saw `service/server-x-default-x-team-1` in the real cluster. My curiosity drives me to try the same from the real cluster, to this service.

```bash
KUBECONFIG="" kubectl -n ns-1 create job --image=curlimages/curl curl -- curl -v http://server-x-default-x-team-1/anything

* Connected to server-x-default-x-team-1 (10.245.98.125) port 80 (#0)
> GET /anything HTTP/1.1
> Host: server-x-default-x-team-1
> User-Agent: curl/7.80.0-DEV
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
{omitted...}
```

It works. The lowest-level stuffs like pod and service "physically" exist in the real cluster and what vcluster does is just sync or we can take it as "sym-link" I believe.

## Exposed vcluster
Tired of port-forwarding holding one of your consoles? Let's try exposing it with a LoadBalancer service as described in vcluster docs:
```bash
$ vcluster create exposed-cluster -n ns-1 --expose 
...
$ kubectl -n ns-1 get service
NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                  AGE
exposed-cluster                            LoadBalancer   10.245.121.213   157.230.192.198   443:32280/TCP            3m12s
exposed-cluster-headless                   ClusterIP      None             <none>            443/TCP                  3m12s
exposed-cluster-node-f72ql                 ClusterIP      10.245.244.3     <none>            10250/TCP                2m26s
kube-dns-x-kube-system-x-exposed-cluster   ClusterIP      10.245.45.180    <none>            53/UDP,53/TCP,9153/TCP   2m26s
kube-dns-x-kube-system-x-team-1            ClusterIP      10.245.32.164    <none>            53/UDP,53/TCP,9153/TCP   23h
server-x-default-x-team-1                  ClusterIP      10.245.98.125    <none>            80/TCP                   43m
team-1                                     ClusterIP      10.245.48.66     <none>            443/TCP                  23h
team-1-headless                            ClusterIP      None             <none>            443/TCP                  23h
team-1-node-9nn58                          ClusterIP      10.245.208.244   <none>            10250/TCP                39m
team-1-node-s2qpb                          ClusterIP      10.245.17.19     <none>            10250/TCP                23h
...
$ vcluster connect exposed-cluster -n ns-1 --server=https://157.230.192.198
...
$ export KUBECONFIG=./kubeconfig.yaml
$ kubectl get ns
NAME              STATUS   AGE
default           Active   9m19s
kube-system       Active   9m19s
kube-public       Active   9m19s
kube-node-lease   Active   9m19s
```

🎉Another brand new cluster ready for another team to mess up with. vcluster allows `--server` so ingress would work too, though we are not going to bother ingress controller in this challenge. 

## Cleanup
To cleanup, it is as easy as:
```bash
vcluster delete team-1 -n ns-1
vcluster delete exposed-cluster -n ns-1
```