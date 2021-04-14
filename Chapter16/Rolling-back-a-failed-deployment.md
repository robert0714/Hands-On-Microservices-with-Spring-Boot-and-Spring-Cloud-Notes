# Rolling back a failed deployment
From time to time, things don't go according to plan, for example, an upgrade of
deployments and pods can fail for various reasons. To demonstrate how to roll back a
failed upgrade, let's try to upgrade to ***v3*** without creating a ***v3*** tag on the Docker image!

Let's try out the following shorthand command to perform the update:
```bash
kubectl set image deployment/product pro=hands-on/product-service:v3
```
Expect to see the following changes reported by the ***kubectl get pod -l app=product -w*** command we launched in the ***Preparing the rolling upgrade*** section:
```
product-7ff56ffc79-dtsfx   0/1     Pending             0          0s
product-7ff56ffc79-dtsfx   0/1     Pending             0          1s
product-7ff56ffc79-dtsfx   0/1     ContainerCreating   0          1s
product-7ff56ffc79-dtsfx   0/1     ErrImageNeverPull   0          3s
```
We can clearly see that the new pod (ending with m2dtn, in my case) has failed to start
because of a problem finding its Docker image (as expected). If we look at the output from
the ***siege*** test tool, no errors are reported, only 200 (OK)! Here, the deployment hangs
since it can't find the requested Docker image, but no errors are affecting end users since
the new pod couldn't even start.

Let's see what history Kubernetes has regarding the product's deployment. Run the
following command:
```
kubectl rollout history deployment product
```
You will receive output similar to the following:
```
$ kubectl rollout history deployment product
deployment.apps/product
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
We can guess that revision 2 is the one with the latest successful deployment, that is, v2 of
the Docker image. Let's check this with the following command:
```
kubectl rollout history deployment product --revision=2
```
In the response, we can see that ***revision #2*** is the one with Docker image ***v2***:
```
$ kubectl rollout history deployment product --revision=2
deployment.apps/product with revision #2
Pod Template:
  Labels:       app=product
        pod-template-hash=57fdd4dd94
  Containers:
   pro:
    Image:      hands-on/product-service:v2
    Port:       80/TCP
    Host Port:  0/TCP
    Limits:
      memory:   400Mi
    Requests:
      memory:   200Mi
    Liveness:   http-get http://:80/actuator/info delay=10s timeout=2s period=10s #success=1 #failure=20
    Readiness:  http-get http://:80/actuator/health delay=10s timeout=2s period=10s #success=1 #failure=3
    Environment Variables from:
      config-client-credentials Secret  Optional: false
    Environment:
      SPRING_PROFILES_ACTIVE:   docker,prod
    Mounts:     <none>
  Volumes:      <none>
```
Let's roll back our deployment to revision=2 with the following command:
```
kubectl rollout undo deployment product --to-revision=2
```
Expect a response that confirms the rollback, like so:
```
$ kubectl rollout undo deployment product --to-revision=2
deployment.apps/product rolled back
```
The ***kubectl get pod -l app=product -w*** command we launched in the ***Preparing the rolling upgrade*** section will report that the new (not working) pod has been removed by the ***rollback*** command:
```
product-7ff56ffc79-dtsfx   0/1     Pending             0          0s
product-7ff56ffc79-dtsfx   0/1     Pending             0          1s
product-7ff56ffc79-dtsfx   0/1     ContainerCreating   0          1s
product-7ff56ffc79-dtsfx   0/1     ErrImageNeverPull   0          3s
product-7ff56ffc79-dtsfx   0/1     Terminating         0          4m55s
product-7ff56ffc79-dtsfx   0/1     Terminating         0          5m11s
product-7ff56ffc79-dtsfx   0/1     Terminating         0          5m11s
```
We can wrap this up by verifying that the current image version is still v2:
```
kubectl get pod -l app=product -o jsonpath='{.items[*].spec.containers[*].image} '
hands-on/product-service:v2
```
## Cleaning up
To delete the resources that we used, run the following commands:
1. Stop the watch command, ***kubectl get pod -l app=product -w***, and the load test program, ***siege***, with ***Ctrl + C***.
2. Delete the namespace:
    ```
    kubectl delete namespace hands-on
    ```
3. Shut down the resource managers that run outside of Kubernetes:
    ```
    eval $(minikube docker-env)
    docker-compose down
    ```
The ***kubectl delete namespace*** command will recursively delete all Kubernetes
resources that existed in the namespace, and the ***docker-compose down*** command will
stop MySQL, MongoDB, and RabbitMQ. With the production environment removed, we
have reached the end of this chapter.
