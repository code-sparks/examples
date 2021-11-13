# Accessing External Services

1. Allow the Envoy proxy to pass requests through to services that are not configured inside the mesh.
2. Configure [service entries](https://archive.istio.io/v1.4/docs/reference/config/networking/service-entry/) to provide controlled access to external services.
3. Manage Traffic for external services.
4. Create Egress gateway for https traffic

## setup

Deploy the sleep app to use as a test source for sending requests, it is located at your istio install folder

```
$ kubectl apply -f sleep.yaml
```

Set the `SOURCE_POD` environment variable to the name of your source pod:

```
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

## 1. Envoy passthrough to external services

setup istio to allow access to external services, if istio  global.outboundTrafficPolicy.mode option set to `ALLOW_ANY`, the Istio proxy lets calls to unknown services pass through, so lets start by checking that this option is set first

```
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```

now lets make a few calls to see that we can send out requests

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
HTTP/2 200
HTTP/2 200
```

## 2. Controlled access to external services

Using Istio `ServiceEntry` configurations, you can access any publicly accessible service from within your Istio cluster. This section shows you how to configure access to an external HTTP service, [httpbin.org](http://httpbin.org/)

### Change to the blocking-by-default policy

To demonstrate the controlled way of enabling access to external services, you need to change the `global.outboundTrafficPolicy.mode` option from the `ALLOW_ANY` mode to the `REGISTRY_ONLY` mode.

```
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
configmap "istio" replaced
```

Make few calls to see if we can send out requests

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
command terminated with exit code 35
command terminated with exit code 35
```

### Access an external HTTP service

Create a `ServiceEntry` to allow access to an external HTTP service:

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

Make a request to the external HTTP service from `SOURCE_POD`:

```
$  kubectl exec -it $SOURCE_POD -c sleep -- curl -I http://httpbin.org | grep  "HTTP/";
HTTP/2 200
```

## 3. Manage traffic to external services

Similar to inter-cluster requests, Istio [routing rules](https://archive.istio.io/v1.4/docs/concepts/traffic-management/#routing-rules) can also be set for external services that are accessed using `ServiceEntry` configurations. In this example, you set a timeout rule on calls to the `httpbin.org` service.

1. From inside the pod being used as the test source, make a *curl* request to the `/delay` endpoint of the httpbin.org external service:

```
$ kubectl exec -it $SOURCE_POD -c sleep sh
$ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
```

The request should return 200 (OK) in approximately 5 seconds.

2. Exit the source pod and use `kubectl` to set a 3s timeout on calls to the `httpbin.org` external service:

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
EOF
```

3. Wait a few seconds, then make the *curl* request again:

```
$ kubectl exec -it $SOURCE_POD -c sleep sh
$ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
504
```

This time a 504 (Gateway Timeout) appears after 3 seconds. Although httpbin.org was waiting 5 seconds, Istio cut off the request at 3 seconds.

## 4. Egress gateway for Https Traffic

In this section you direct HTTPS traffic (TLS originated by the application) through an egress gateway. You need to specify port 443 with protocol `TLS` in a corresponding `ServiceEntry`, an egress `Gateway` and a `VirtualService`.

1. Define a `ServiceEntry` for `edition.cnn.com`::

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 443
    name: tls
    protocol: TLS
  resolution: DNS
EOF
```

2. Verify that your `ServiceEntry` was applied correctly by sending an HTTPS request to https://edition.cnn.com/politics.

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
...
Content-Length: 151654
...
```

3. Create an egress `Gateway` for *edition.cnn.com*, a destination rule and a virtual service to direct the traffic through the egress gateway and from the egress gateway to the external service.

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    hosts:
    - edition.cnn.com
    tls:
      mode: PASSTHROUGH
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-cnn
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: cnn
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-cnn-through-egress-gateway
spec:
  hosts:
  - edition.cnn.com
  gateways:
  - mesh
  - istio-egressgateway
  tls:
  - match:
    - gateways:
      - mesh
      port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: cnn
        port:
          number: 443
  - match:
    - gateways:
      - istio-egressgateway
      port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 443
      weight: 100
EOF

```

4. Send an HTTPS request to https://edition.cnn.com/politics. The output should be the same as before.

```
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
...
Content-Length: 151654
...
```

5. Check the log of the egress gateway’s proxy. If Istio is deployed in the `istio-system` namespace, the command to print the log is:

```
$ kubectl logs -l istio=egressgateway -n istio-system
```

