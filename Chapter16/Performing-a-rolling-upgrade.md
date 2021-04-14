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
HTTP/1.1 200   0.10 secs:    1353 bytes ==> GET  /actuator/health
HTTP/1.1 200   0.06 secs:    1353 bytes ==> GET  /actuator/health
HTTP/1.1 200   0.07 secs:    1353 bytes ==> GET  /actuator/health
```
The interesting part in the response is the HTTP status code, which we expect to be 200 at
all times.
Also, monitor changes to the state of the product pods with the following command:
```
kubectl get pod -l app=product -w
```

## Upgrading the product service from v1 to v2
To upgrade the product service, edit the ***kubernetes/services/overlays/prod/product-prod.yml*** file and change ***image:hands-on/product-service:v1*** to ***image: hands-on/product-service:v2***.
Apply the update with the following command:
```
kubectl apply -k kubernetes/services/overlays/prod
```
Expect a response from the command that reports that most of the objects are left
unchanged, except for the product deployment that should be reported to be updated
to ***deployment.apps/product configured*** .

- Kubernetes comes with some shorthand commands. For example, ***kubectl set image deployment/product pro=hands-on/product-service:v2*** can be used to perform the same update that we did by updating the definitions file and running the ***kubectl apply*** command. A major benefit of using the kubectl apply command
is that we can keep track of the changes by pushing the changes in the source code to a version control system such as Git. This is very important if we want to be able to handle our infrastructure as code. When playing around with a Kubernetes cluster, only use it to test shorthand commands, as this can be very useful.

In the output from the ***kubectl get pod -l app=product -w*** command we launched in the ***Preparing the rolling upgrade*** section, we will see some action occurring. Take a look at the following screenshot:
```
$ kubectl get pod -l app=product -w
NAME                       READY   STATUS    RESTARTS   AGE
product-66ccc8f4b9-jfxhd   1/1     Running   0          39m
product-57fdd4dd94-wg25r   0/1     Pending   0          0s
product-57fdd4dd94-wg25r   0/1     Pending   0          0s
product-57fdd4dd94-wg25r   0/1     ContainerCreating   0          0s
product-57fdd4dd94-wg25r   0/1     Running             0          2s
product-57fdd4dd94-wg25r   1/1     Running             0          24s
product-66ccc8f4b9-jfxhd   1/1     Terminating         0          44m
product-66ccc8f4b9-jfxhd   0/1     Terminating         0          44m
product-66ccc8f4b9-jfxhd   0/1     Terminating         0          45m
product-66ccc8f4b9-jfxhd   0/1     Terminating         0          45m
```
Here, we can see how the existing pod (jfxhd) initially reported that it was up and running
and also reported to be healthy when a new pod was launched (wg25r). After a while (24s,
in my case), it is reported as up and running as well. During a certain time period, both
pods will be up and running and processing requests. After a while, the first pod is
terminated (44 minutes, in my case).

When looking at the ***siege*** output, we can sometimes find a few errors being reported in
terms of the ***503*** service unavailable errors:
```bash
HTTP/1.1 200   0.05 secs:    1353 bytes ==> GET  /actuator/health
HTTP/1.1 503   0.08 secs:    1868 bytes ==> GET  /actuator/health
HTTP/1.1 503   0.08 secs:    1868 bytes ==> GET  /actuator/health
HTTP/1.1 200   0.05 secs:    1353 bytes ==> GET  /actuator/health
```
This typically happens when the old pod is terminated. Before the old pod is reported
unhealthy by the readiness probe, it can receive a few requests during its termination, that
is, when it is no longer capable of serving any requests.

- In Chapter 18, Using a Service Mesh to Improve Observability and
Management, we will see how we can set up routing rules that move traffic
in a smoother way from an old pod to a newer one without causing 503
errors. We will also see how we can apply retry mechanisms to stop
temporary failures from reaching an end user.

Wrap this up by verifying that the pod is using the new v2 version of the Docker image:
```
kubectl get pod -l app=product -o jsonpath='{.items[*].spec.containers[*].image} '
```

The expected output reveals that v2 of the Docker image is in use:
```
$ kubectl get pod -l app=product -o jsonpath='{.items[*].spec.containers[*].image} '
hands-on/product-service:v2
```
After performing this upgrade, we can move on to learning what happens when things fail.
In the next section, we will see how we can roll back a failed deployment.
