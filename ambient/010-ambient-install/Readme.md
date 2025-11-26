## 1. About OpenShift Service Mesh Ambient Mode

Istio ambient mode introduces a new way to manage service mesh without using traditional sidecar proxies. The biggest change is how it separates network traffic processing into two distinct layers, which is the core architectural difference. This architecture simplifies networking, reduces resource usage, and improves security while supporting the same service mesh use cases.

Ambient mode uses a different data plane architecture that splits traffic processing between a per-node Layer 4 (L4) proxy called Ztunnel (Zero-Trust Tunnel) and an optional Layer 7 (L7) proxy called waypoint proxy.

### Installing OpenShift Service Mesh Ambient Mode

Navigate to the directory: `010-ambient-install`

Create the namespaces `istio-system`, `istio-cni` and `ztunnel`:

```sh
$ oc apply -f 01-ns-create.yaml
```

Now install the controlplane by applying the `Istio`, `IstioCNI` resource with profile ambient. We also use discovery selectors to scope the mesh:

```sh
$ oc apply -f 02-istio-control-plane.yaml
```

Next install the dataplane by applying the `ZTunnel` resource:

```sh
$ oc apply -f 03-istio-dataplane.yaml
```

### Verify the installation

```sh
$ oc get pods -n istio-system

NAME                      READY   STATUS    RESTARTS   AGE
istiod-69b5fc4898-b7x4x   1/1     Running   0          82s
```

```sh
$ oc get daemonset -n istio-cni

NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
istio-cni-node   3         3         3       3            3           kubernetes.io/os=linux   110s
```

```sh
$ oc get daemonset -n ztunnel

NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
ztunnel   3         3         3       3            3           kubernetes.io/os=linux   2m24s
```
