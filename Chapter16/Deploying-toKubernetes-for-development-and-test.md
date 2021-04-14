<!-- MarkdownTOC -->
- [Deploying to Kubernetes for development and test](#deploying-to-kubernetes-for-development-and-test)
    - [1. Building Docker images](#1-building-docker-images)
    - [2. Deploying to Kubernetes](#2-changes-in-the-source-code)
    - [3. Changes in the test script for use with Kubernetes](#3-changes-in-the-test-script-for-use-with-kubernetes)
      - [3.1. Reaching the internal actuator endpoint using Docker Compose](##-31-reaching-the-internal-actuator-endpoint-using-docker-compose)
      - [3.2. Reaching the internal actuator endpoint using Kubernetes](##-32-reaching-the-internal-actuator-endpoint-using-kubernetes)
      - [3.3. Choosing between Docker Compose and Kubernetes](##-33-choosing-between-docker-compose-and-kubernetes)
    - [4. Testing the deployment](#4-Testing-the-deployment)

<!-- /MarkdownTOC -->

# Deploying to Kubernetes for development and test

page 435

## 1. Building Docker images

In our case, where we have a local single node cluster, we can
shortcut this process by pointing our Docker client to the Docker engine in ***Minikube*** and
then run the ***docker-compose build*** command.

From Kubernetes 1.15, this is very simple. Just change the code and
rebuild the Docker image, for example, using the build command that's
described here. Then, update a pod with the ***kubectl rollout restart*** command.
For example, if the ***product*** service has been updated, run the ***kubectl rollout restart deploy product*** command.

You can build Docker images from source as follows:
```bash
cd $BOOK_HOME/Chapter16
eval $(minikube docker-env)
./gradlew build && docker-compose build
```

## 2. Deploying to Kubernetes

Create a namespace, ***hands-on***, and set it as the default namespace for kubectl:
```bash
kubectl create namespace hands-on
kubectl config set-context $(kubectl config current-context) --namespace=hands-on
```
The credentials for
connecting to the configuration server and the encryption key will be stored in two secrets,
one for the configuration server and one for its clients.

To check this, perform the following steps:

1. Create the config map for the configuration repository based on the files in the
config-repo folder with the following command:
    ```
    kubectl create configmap config-repo --from-file=config-repo/ --save-config
    ```
2. Create the secret for the configuration server with the following command:
    ```
    kubectl create secret generic config-server-secrets \
    --from-literal=ENCRYPT_KEY=my-very-secure-encrypt-key \
    --from-literal=SPRING_SECURITY_USER_NAME=dev-usr \
    --from-literal=SPRING_SECURITY_USER_PASSWORD=dev-pwd \
    --save-config
    ```
3. Create the secret for the clients of the configuration server with the following command:    
    ```
    kubectl create secret generic config-client-credentials \
    --from-literal=CONFIG_SERVER_USR=dev-usr \
    --from-literal=CONFIG_SERVER_PWD=dev-pwd --save-config
    ```
    Since we have just entered commands that contain sensitive information
in clear text, for example, passwords and an encryption key, it is a good
idea to clear the history command. To clear the ***history*** command both
in memory and on disk, run the ***history -c; history -w*** command.

4. To avoid a slow deployment due to Kubernetes downloading Docker images
(potentially causing the liveness probes we described previously to restart our
pods), run the following docker pull commands to download the images:
    ```
    docker pull mysql:5.7
    docker pull mongo:3.6.9
    docker pull rabbitmq:3.7.8-management
    docker pull openzipkin/zipkin:2.12.9
    ```
5. Deploy the microservices for the development environment, based on the ***dev***
overlay, using the ***-k*** switch to activate Kustomize, as described previously:   
    ```
    kubectl apply -k kubernetes/services/overlays/dev
    ```
6. Wait for the deployments and their pods to be up and running by running the
following command:
    ```
    kubectl wait --timeout=600s --for=condition=ready pod --all
    ```
    Expect each command to respond with ***deployment.extensions/...condition met.*** ... will be replaced with the name of the actual deployment.
7. To see the Docker images that are used for development, run the following
command:
    ```
    kubectl get pods -o json | jq .items[].spec.containers[].image
    ```
    
We are now ready to test our deployment!
But before we can do that, we need to go through changes that are required in the test script
for use with Kubernetes.

## 3. Changes in the test script for use with Kubernetes

To test the deployment we will, as usual, run the test script, that is, ***test-em-all.bash***. To
work with Kubernetes, the circuit breaker tests have been slightly modified. Take a look at
the ***testCircuitBreaker()*** function for more details. The circuit breaker tests call
the actuator endpoints on the ***product-composite*** service to check their health state
and get access to circuit breaker events. The ***actuator*** endpoints are not exposed
externally, so the test script needs to use different techniques to access the internal
endpoints when using Docker Compose and Kubernetes:

1. When using Docker Compose, the test script will launch a Docker container using a plain ***docker run*** command that calls the ***actuator*** endpoints from the inside of the network created by Docker Compose.  
2. When using Kubernetes, the test script will launch a Kubernetes pod that it can use to run the corresponding commands inside Kubernetes.
Let's see how this is done when using Docker Compose and Kubernetes.

### 3.1. Reaching the internal actuator endpoint using Docker Compose

The base command that's defined for Docker Compose is as follows:
```
EXEC="docker run --rm -it --network=my-network alpine"
```
Note that the container will be killed using the --rm switch after each execution of a test
command.

### 3.2. Reaching the internal actuator endpoint using Kubernetes

Since launching a pod in Kubernetes is slower than starting a container, the test script will
launch a single pod, ***alpine-client***. The pod will be launched at the start
of the ***testCircuitBreaker()*** function, and the tests will use the ***kubectl exec***
command to run the test commands in this pod. This will be much faster than creating and
deleting a pod for each test command.

Launching the single pod is handled at the beginning of the testCircuitBreaker()
function:

```bash
        echo "Restarting alpine-client..."
        local ns=$NAMESPACE
        if kubectl -n $ns get pod alpine-client > /dev/null ; then
            kubectl -n $ns delete pod alpine-client --grace-period=1
        fi
        kubectl -n $ns run --restart=Never alpine-client --image=alpine --command -- sleep 600
        echo "Waiting for alpine-client to be ready..."
        kubectl -n $ns wait --for=condition=Ready pod/alpine-client

        EXEC="kubectl -n $ns exec alpine-client --"
```
At the end of the circuit breaker tests, the pod is deleted by using the following command:
```bash
kubectl -n $ns delete pod alpine-client --grace-period=1
```

### 3.3. Choosing between Docker Compose and Kubernetes

To make the test script work with both Docker Compose and Kubernetes, it assumes that
Docker Compose will be used if the ***HOST*** environment variable is set to ***localhost***;
otherwise, it assumes that Kubernetes will be used. See the following code:
```
# Assume we are using Docker Compose if we are running on localhost, otherwise Kubernetes 
    if [ "$HOST" = "localhost" ]
    then
        EXEC="docker run --rm -it --network=my-network alpine"
    else
        echo "Restarting alpine-client..."
       ...(ommit)
        EXEC="kubectl -n $ns exec alpine-client --"
    fi

```
The default value for the HOST environment variable in the test script is localhost.

Once the ***EXEC*** variable has been set up, depending on whether the tests are running on
Docker Compose or on Kubernetes, it is used in the ***testCircuitBreaker()*** test function.
The test starts by verifying that the circuit breaker is closed with the following statement:
```bash
 # First, use the health - endpoint to verify that the circuit breaker is closed
    assertEqual "CLOSED" "$($EXEC wget product-composite/actuator/health -qO - | jq -r .components.circuitBreakers.details.product.details.state)"

```
A final change in the test script occurs because our services are now reachable on
the ***80*** port inside the cluster; that is, they are no longer on the ***8080*** port.

## 4. Testing the deployment

When launching the test script, we have to give it the address of the host that runs
Kubernetes, that is, our Minikube instance, and the external port where our gateway service
listens for external requests. The ***minikube ip*** command can be used to find the IP
address of the Minikube instance and, as mentioned in the ***Setting up common definitions in the base folder*** section, we have assigned the external ***NodePort 31443*** (page 434) to the gateway service.

Start the tests with the following command:
```
HOST=$(minikube ip) PORT=31443 ./test-em-all.bash
```
In the output from the script we will see how the IP address of the Minikube instance is
used and also how the alpine-client pod is created and destroyed:

Before we move on and look at how to set up a corresponding environment for staging and
production use, let's clean up what we have installed in the development environment to
preserve resources in the Kubernetes cluster. We can do this by simply deleting the
namespace. Deleting the namespace will recursively delete the resources that exist in the
namespace.

Delete the namespace with the following command:
```
kubectl delete namespace hands-on
```
With the development environment removed, we can move on and set up an environment
targeting staging and production.
