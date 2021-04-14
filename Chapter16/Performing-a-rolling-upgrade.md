# Performing a rolling upgrade
<!-- MarkdownTOC -->
- [Performing a rolling upgrade](#performing-a-rolling-upgrade)
    - [Preparing the rolling upgrade](#preparing-the-rolling-upgrade)
    - [Upgrading the product service from v1 to v2](#upgrading-the-product-service-from-v1-to-v2)   

<!-- /MarkdownTOC -->
Performing a rolling upgrade means that Kubernetes first starts the new version of the microservice in a new
pod, and when it reports as being healthy, Kubernetes will terminate the old one. This
ensures that there is always a pod up and running, ready to serve incoming requests during
the upgrade. 

A prerequisite for a rolling upgrade to work is that the upgrade is backward
compatible, both in terms of APIs and message formats that are used to communicate with
other services and database structures. If the new version of the microservice requires
changes to either the external APIs, message formats, or database structures that the old
version can't handle, a rolling upgrade can't be applied. A deployment object is configured
to perform any updates as a rolling upgrade by default.

To try this out, we will create a v2 version of the Docker image for the product service and
then start up a test client, ***siege***, that will submit one request per second during the rolling
upgrade. The assumption is that the test client will report 200 (OK) for all the requests that
it sends during the upgrade.
## Preparing the rolling upgrade

## Upgrading the product service from v1 to v2
To prepare for the rolling upgrade, first, verify that we have the v1 version of the product pod deployed:
```bash
kubectl get pod -l app=product -o jsonpath='{.items[*].spec.containers[*].image} '
```
The expected output should reveal that v1 of the Docker image is in use:
```bash
hands-on/product-service:v1
```
Create a v2 tag on the Docker image for the product service with the following command:
```bash
docker tag hands-on/product-service:v1 hands-on/product-service:v2
```
- To try out a rolling upgrade from a Kubernetes perspective, we don't need
to change any code in the ***product*** service. Deploying a Docker image
with another tag than the existing one will start up a rolling upgrade.

To be able to observe whether any downtime occurs during the upgrade, we will start a
low volume load test using siege. The following command starts a load test that simulates
one user (***-c1***) that submits one request per second on average (***-d1***):
Mac / Linux  
```
siege https://$(minikube ip):31443/actuator/health -c1 -d1
```
windows(https://github.com/ewwink/siege-windows)
C:\Users\YOUR-USERNAME\.siegerc (Windows )
```
verbose = true
color = on
quiet = false
show-logfile = true
logging = false
limit = 3000
protocol = HTTP/1.1
chunked = true
cache = false
concurrent = 3000
connection = close
```
Since the test calls the gateways health endpoint, it verifies that all the services are healthy.

You should receive an output that looks similar to the following screenshot:

```
```
The interesting part in the response is the HTTP status code, which we expect to be 200 at
all times.
Also, monitor changes to the state of the product pods with the following command:
```
kubectl get pod -l app=product -w
```
