## 7. Implementing Resilience

In this module we focus on **resilience**: the ability of a system to continue operating correctly even when parts of it fail. In distributed and microservices-based architectures, failures are not an exception, they are expected. Networks are unreliable, services can become slow or unavailable, and traffic patterns can change suddenly. A resilient system absorbs these problems and degrades gracefully instead of failing completely.

Resilience is important because it:

* Improves application availability and user experience
* Prevents cascading failures between services
* Makes systems more predictable and easier to operate
* Enables safer deployments and faster recovery from incidents

### About Resilience with Service Mesh

Istio and OpenShift Service Mesh provides powerful built-in features to implement resilience at the service mesh level, without changing application code. Key features include:

* **Retries**: Automatically retry failed requests based on policies
* **Timeouts**: Prevent requests from hanging indefinitely
* **Circuit breakers**: Stop sending traffic to unhealthy services to avoid overload
* **Fault injection**: Simulate failures and latency to test resilience
* **Load balancing strategies**: Distribute traffic intelligently across instances
* **Outlier detection**: Automatically eject misbehaving service instances

With Istio, resilience becomes a platform capability rather than something each application must implement on its own. This module will show how to use these features to build systems that are more robust, predictable, and production-ready.

### 7.1 Fault injection

In this module we are wanting to make the travel application more resilient. 

Fault injection lets you create failures on purpose in a controlled and repeatable way. Instead of waiting for real outages, slow networks, or crashing pods, you can simulate them and see how your system behaves. That is the key to testing resilience: you verify that your system reacts correctly before something breaks in production.

We are looking at the following request path:

External traffic -> Ingress Gateway -> travel-portal(s) -> travels API -> cars service

Injecting faults is not possible with `HTTPRoute`, so we are using a `VirtualService` resource instead.
In this case we configure a 10s delay for 100% of the calls to the cars service:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: cars-inject-delay
  namespace: travel-agency
spec:
  hosts:
  - cars
  http:
  - fault:
      delay:
        fixedDelay: 10s
        percentage:
          value: 100.0     
    route:
    - destination:
        host: cars
```

```sh
oc apply -f 01-fault-injection.yaml
```

Next, test the behavior by sending requests to the travels API, which in turn calls the cars service:

```sh
oc exec $(oc get pod -l app=cars -n travel-agency -o jsonpath='{.items[0].metadata.name}') -n travel-agency -- curl -sSv travels.travel-agency.svc.cluster.local:8000/travels/Madrid
```

After a 10 seconds delay we are eventually getting the response. Please notice also the `x-envoy-upstream-service-time` header.

```
< HTTP/1.1 200 OK
< content-type: application/json
< date: Fri, 16 Jan 2026 16:26:45 GMT
< content-length: 507
< x-envoy-upstream-service-time: 10018
< server: istio-envoy
< x-envoy-decorator-operation: travels.travel-agency.svc.cluster.local:8000/*
< 
{ [data not shown]
* Connection #0 to host travels.travel-agency.svc.cluster.local left intact
{"city":"Madrid","coordinates":null,"createdAt":"2026-01-16T16:26:35Z","status":"Valid","flights":[{"airline":"Red Airlines","price":1020},{"airline":"Blue Airlines","price":370},{"airline":"Green Airlines","price":320}],"hotels":[{"hotel":"Grand Hotel Madrid","price":600},{"hotel":"Little Madrid Hotel","price":120}],"cars":[{"carModel":"Sports Car","price":1100},{"carModel":"Economy Car","price":340}],"insurances":[{"company":"Yellow Insurances","price":327},{"company":"Blue Insurances","price":76}]}
```

### 7.2 Request timeouts

Istio configures no timeouts by default.
In this step we are going to configure a timeout configuration:

- Every request to travels has a maximum total lifetime of 3 seconds
(timeout: 3s)
- If a request fails or times out, Istio will retry it up to 2 times
(attempts: 2 → 1 initial try + 2 retries = 3 total attempts)
- Each individual attempt is allowed to run for at most 1 second
(perTryTimeout: 1s)

Retries are triggered on:
- HTTP 5xx responses
- Connection failures
- Stream resets
- Timeouts

This VirtualService enforces fast failure and controlled retries for the travels service, making client behavior predictable and resilient without changing application code.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: travels-request-timeout
  namespace: travel-agency
spec:
  hosts:
  - travels
  http:
  - timeout: 3s
    retries:
      attempts: 2          
      perTryTimeout: 1s
      retryOn: 5xx,connect-failure,refused-stream,reset,timeout  
    route:
    - destination:
        host: travels 
```

```sh
oc apply -f 02-request-timeout.yaml
```

Now let us test this by sending again a request to the travels API:

```sh
oc exec $(oc get pod -l app=cars -n travel-agency -o jsonpath='{.items[0].metadata.name}') -n travel-agency -- curl -sSv travels.travel-agency.svc.cluster.local:8000/travels/Madrid
```

You'll see that the request successfully reached the travels service, but the service did not respond in time. Envoy waited for a response from the upstream service, but the configured timeout expired.

```
* About to connect() to travels.travel-agency.svc.cluster.local port 8000 (#0)
*   Trying 172.30.150.117...
* Connected to travels.travel-agency.svc.cluster.local (172.30.150.117) port 8000 (#0)
> GET /travels/Madrid HTTP/1.1
> User-Agent: curl/7.29.0
> Host: travels.travel-agency.svc.cluster.local:8000
> Accept: */*
> 
upstream request timeout< HTTP/1.1 504 Gateway Timeout
```

Clean up:

```sh
oc delete -f 01-fault-injection.yaml 02-request-timeout.yaml
```

### 7.3 Circuit Breaking

Circuit breaking is a resilience pattern that stops sending traffic to a service that is unhealthy or overloaded.

Instead of constantly retrying and making the situation worse, the circuit breaker “opens” and fails fast. This protects both the failing service and the rest of the system from cascading failures.

To test circuit breaking, we use Fortio. 
Fortio is a lightweight load testing and traffic generation tool, commonly used with Istio. 

**Step 1:** 
Create a fortio deployment in the `travel-agency` namespace:

```sh
oc apply -n travel-agency -f https://raw.githubusercontent.com/istio/istio/release-1.26/samples/httpbin/sample-client/fortio-deploy.yaml
```

**Step 2:** 
When fortio is ready, create an initial load test:

That command basically does the following: Send 100 requests to the cars service using 3 concurrent connections and 10 request per second, and show minimal output.

```sh
oc -n travel-agency exec deploy/fortio-deploy -- fortio load -c 3 -qps 10 -n 100 \
  -quiet http://cars:8000/cars/Madrid
```

The result should look similar to this:

```
Fortio 1.69.5 running at 10 queries per second, 6->6 procs, for 100 calls: http://cars:8000/cars/Madrid
Aggregated Function Time : count 100 avg 0.0096797149 +/- 0.002229 min 0.006753042 max 0.017862949 sum 0.967971488
# target 50% 0.0091
# target 75% 0.0105833
# target 90% 0.0124
# target 99% 0.017242
# target 99.9% 0.0178009
Error cases : count 0 avg 0 +/- 0 min 0 max 0 sum 0
# Socket and IP used for each connection:
[0]   1 socket used, resolved to 172.30.123.174:8000, connection timing : count 1 avg 0.000122223 +/- 0 min 0.000122223 max 0.000122223 sum 0.000122223
[1]   1 socket used, resolved to 172.30.123.174:8000, connection timing : count 1 avg 0.000176919 +/- 0 min 0.000176919 max 0.000176919 sum 0.000176919
[2]   1 socket used, resolved to 172.30.123.174:8000, connection timing : count 1 avg 0.000660092 +/- 0 min 0.000660092 max 0.000660092 sum 0.000660092
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
172.30.123.174:8000: 3
Code 200 : 100 (100.0 %)
All done 100 calls (plus 0 warmup) 9.680 ms avg, 9.8 qps
```

You will notice that all the request `Code 200 : 100 (100.0 %)` where successful.

**Step 3:** 
Next we artificially 'slow' down the cars service by configuring the waypoint with low thresholds.

We configure a connection pool with a single connection, a maximum of one request per connection, and a pending requests queue with a maximum size of one:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: cars-low-threshold-policy
  namespace: travel-agency
spec:
  host: cars
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
```

```sh
oc apply -f 03-cars-low-threshold-policy.yaml
```

**Step 4:**
We can trip the circuit breaker by sending in more concurrent requests:

```sh
oc exec deploy/fortio-deploy -- fortio load -c 3 -qps 10 -n 100 \
  -quiet http://cars:8000/cars/Madrid
```

you should see a result similar to this:

```
Code 200 : 72 (72.0 %)
Code 503 : 28 (28.0 %)
All done 100 calls (plus 0 warmup) 12.832 ms avg, 9.8 qps
```

Let us also have a look at the logs. You'll see a bunch of 503 responses:

```sh
oc logs -n travel-agency -f deploy/waypoint-travel-agency
```

```
...
[2026-01-17T10:31:28.636Z] "GET /cars/Madrid HTTP/1.1" 503 UO upstream_reset_before_response_started{overflow} - "-" 0 81 0 - "-" "fortio.org/fortio-1.69.5" "e19aefd5-737d-441a-8340-1d96ea362d09" "cars:8000" "-" inbound-vip|8000|http|cars.travel-agency.svc.cluster.local - 172.30.123.174:8000 10.130.0.125:38310 - default
...
```

Envoy proxy has response flags for specific events that occurred when handling requests. 
The `UO (UpstreamOverflow)` flag is indicating a circuit breaking.
Istio prevented sending requests to the service.

### 7.4 Outlier Detecting (Optional Challenge)

Outlier detection is a mechanism that automatically removes unhealthy service instances (pods) from load balancing when they start failing.
If a pod returns too many errors, Istio marks it as an “outlier” and temporarily stops sending traffic to it. 
After some time, it is tried again to see if it has recovered.

Outlier detection protects your system by isolating bad pods before they can impact the whole application.

This policy means:
- If a cars pod returns 2 consecutive 5xx errors, it is considered unhealthy.
- Istio will eject it from load balancing for 15 seconds.
- Up to 100% of the pods can be ejected if they all misbehave.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: cars-outlier-detection-policy
  namespace: travel-agency
spec:
  host: cars
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 2
      baseEjectionTime: 15s
      maxEjectionPercent: 100
```

```sh
oc apply -f 04-cars-outlier-detection-policy.yaml
```

Your goal is to prove that this outlier detection policy is working:

Design and execute a test that:
- Causes one or more cars pods to start returning HTTP 5xx errors.
- Generates enough traffic to trigger the outlier detection.

Observes that:
- Failing pods are removed from traffic.
- Requests are no longer sent to the unhealthy pods.
- After ~15 seconds, Istio tries the pods again.

Hints:
- Make sure the cars service has multiple replicas.
- Find a way to make only some pods fail (not all).

Watch:
- Access logs
- Pod traffic in Kiali
- Error rates over time