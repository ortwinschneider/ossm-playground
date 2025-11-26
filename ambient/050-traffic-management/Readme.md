## 5. Traffic Management

To gain advanced traffic control, you **must deploy a Waypoint Proxy** for the workload's namespace. The traffic is then routed from the Ztunnel, through the Waypoint Proxy, and finally to the destination workload.

The following section describes some advanced traffic control capabilities:

- **Traffic Splitting**: Uses `Waypoint` Proxy
  - Percentage-based routing (e.g., 90% to V1, 10% to V2).
- **Header-Based Routing**:	Uses `Waypoint` Proxy	
  - Route traffic based on HTTP headers 
- **Traffic Mirroring**: Uses `Waypoint` Proxy
  - Shadowing a percentage of live traffic to a test service.
- **Fault Injection:** Uses `Waypoint` Proxy
  - Injecting artificial delays or HTTP failure codes for resilience testing.
- **L4 Load Balancing**: Uses `Ztunnel` Only 
  - Basic connection-level balancing (default behavior).
- **Timeouts/Retries**: Uses `Waypoint` Proxy
  - Configuring application-level request timeouts and retries on failure.

### 5.1 Create Waypoint Proxies

```sh
oc apply -f 01-waypoints-create.yaml
```

### 5.2 Enroll the travel application namespaces to use waypoints

```sh
oc apply -f 02-ns-use-waypoints.yaml
```

After enrolling the namespace, requests from any pods using the ambient data plane to services in bookinfo will route through the waypoint for L7 processing and policy enforcement.

Confirm that the waypoint proxy is used by all the services in the travel-portal and travel-agency namespaces:

```sh
istioctl ztunnel-config svc --namespace ztunnel
```

### 5.3 High Availability of Waypoint Proxies

```sh
oc apply -f 03-waypoint-hpa.yaml
```

### 5.4 Pod Disruption Bucket for Waypoint Proxies

```sh
oc apply -f 04-waypoint-pdb.yaml
```