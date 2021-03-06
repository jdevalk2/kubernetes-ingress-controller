1. Install [`minikube`](https://github.com/kubernetes/minikube)

Minikube is a tool that makes it easy to run Kubernetes locally.
Minikube runs a single-node Kubernetes cluster inside a VM on your laptop
for users looking to try out Kubernetes or develop with it day-to-day.

2. Start `minikube`

```bash
minikube start
```

It will take few minutes to get all resources provisioned.

```bash
kubectl get nodes
```

3. Deploy Kong (0.13.0 or higher is required)

**Deploy the required components**

```bash
curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml \
  | kubectl create -f -
```

This commands create:

```bash

namespace "kong" created
customresourcedefinition "kongplugins.configuration.konghq.com" created
customresourcedefinition "kongconsumers.configuration.konghq.com" created
customresourcedefinition "kongcredentials.configuration.konghq.com" created
service "postgres" created
statefulset "postgres" created
serviceaccount "kong-serviceaccount" created
clusterrole "kong-ingress-clusterrole" created
role "kong-ingress-role" created
rolebinding "kong-ingress-role-nisa-binding" created
clusterrolebinding "kong-ingress-clusterrole-nisa-binding" created
service "kong-ingress-controller" created
deployment "kong-ingress-controller" created
service "kong-proxy" created
deployment "kong" created
```

*Note:* this process could take up to five minutes the first time

**Create environment variables**

```bash
export KONG_ADMIN_PORT=$(minikube service -n kong kong-ingress-controller --url --format "{{ .Port }}")
export KONG_ADMIN_IP=$(minikube service   -n kong kong-ingress-controller --url --format "{{ .IP }}")

export PROXY_IP=$(minikube   service -n kong kong-proxy --url --format "{{ .IP }}" | head -1)
export HTTP_PORT=$(minikube  service -n kong kong-proxy --url --format "{{ .Port }}" | head -1)
export HTTPS_PORT=$(minikube service -n kong kong-proxy --url --format "{{ .Port }}" | tail -1)
```

**Deploy a dummy application**

```bash
curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/manifests/dummy-application.yaml \
  | kubectl create -f -
```

This application just returns information about the pod and details from the HTTP request.
This is an example of the output:

```console
Hostname: http-svc-7dd9588c5-gmbvh

Pod Information:
	node name:	minikube
	pod name:	http-svc-7dd9588c5-gmbvh
	pod namespace:	default
	pod IP:	172.17.0.7

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=127.0.0.1
	method=GET
	real path=/
	query=
	request_version=1.1
	request_uri=http://localhost:8080/

Request Headers:
	accept=*/*
	host=localhost:8080
	user-agent=curl/7.47.0

Request Body:
	-no body in request-
```

4. Create an Ingress rule

```bash
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: foo-bar
spec:
  rules:
  - host: foo.bar
    http:
      paths:
      - path: /
        backend:
          serviceName: http-svc
          servicePort: 80
" | kubectl create -f -
```

Test the Ingress rule running:

```bash
http ${PROXY_IP}:${HTTP_PORT} Host:foo.bar
```

5. Create Ingress rule with TLS section (HTTPS)

a. Create a secret with a SSL certificate

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=secure-foo-bar/O=konghq.org"
```

```bash
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

b. Create an Ingress rule with a TLS section

```bash
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: secure-foo-bar
spec:
  tls:
  - hosts:
    - secure.foo.bar
    secretName: tls-secret
  rules:
  - host: secure.foo.bar
    http:
      paths:
      - path: /
        backend:
          serviceName: http-svc
          servicePort: 80
" | kubectl create -f -
```

Test the Ingress rule running:

```bash
curl -v -k --resolve secure.foo.bar:${HTTPS_PORT}:${PROXY_IP} https://secure.foo.bar:${HTTPS_PORT}
```

**Logs**

Using `kubectl logs` it is possible to follow the interaction of the Ingress controller and Kong.
```
kubectl logs -n kong --selector="app=ingress-kong" -c ingress-controller
```

**Scaling a deployment**

One of the main features provided by an Ingress controller is the ability to react to changes in the Kubernetes cluster.
This means if we scale a deployment or a pod dies we need to update the Kong configuration (a Target in this case)
```bash
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/upstreams/default.http-svc.80/targets
```

To see this we can execute:

```bash
kubectl scale deployment http-svc --replicas=5
```

After a second we should have more targets (pods)

```bash
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/upstreams/default.http-svc.80/targets
```

If we scale down the number of replicas or a pod is killed the Kong target list will be updated

**Configure a Kong plugin using annotations in Ingress**

a. Create a Kong plugin running:

```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: add-ratelimiting-to-route
config:
  hour: 100
  limit_by: ip
  second: 10
" | kubectl create -f -
```

b. Check the plugins was created correctly running:

```bash
kubectl get kongplugins
NAME                        AGE
add-ratelimiting-to-route   16s
```

c. Patch the Ingress to add the annotation for the rate-limiting plugin

```bash
kubectl patch ingress foo-bar \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com":"add-ratelimiting-to-route\n"}}}'
```

d. Check the plugin was created in Kong running:

```bash
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/plugins
```

d. Verify the plugin was configured making a request and checking the `X-RateLimit-Limit-*` headers are present in the response

```bash
http ${PROXY_IP}:${HTTP_PORT} Host:foo.bar
```

**Configure a Kong plugin using annotations in Kubernetes Services**

Using annotations we can decide where we want to apply plugins. We have two options:

- Annotations in an Ingress Kong will apply the plugin/s to the Route
- Annotations in a Kubernetes Service will apply the plugin/s to the Kong Service

*Note:* Annotations on a Service takes precedence over annotations on an Ingress

```bash
kubectl patch svc http-svc \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com": "add-ratelimiting-to-route\n"}}}'
```

This change should remove the plugin from the route and create a new plugin using the Kong Service as a source

```bash
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/plugins
```

**Delete Kong**

```bash
  kubectl delete namespace kong
```
