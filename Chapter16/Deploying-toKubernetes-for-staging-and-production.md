# Deploying to Kubernetes for staging and production

<!-- MarkdownTOC -->
- [Deploying to Kubernetes for staging and production](#deploying-to-kubernetes-for-staging-and-production)
    - [Changes in the source code](#changes-in-the-source-code)
    - [Deploying to Kubernetes](#deploying-to-kubernetes)   

<!-- /MarkdownTOC -->
A staging environment is used for performing ***quality assurance (QA)***
and ***user acceptance tests (UAT)*** as the last step before taking a new release into
production.

When deploying to an environment for staging or production, there are a number of
changes required compared to when deploying for development or tests:

- ***Resource managers should run outside of the Kubernetes cluster***
    - It is technically feasible to run databases and queue managers for production use on
Kubernetes as stateful containers using StatefulSets
and PersistentVolumes.  Instead, I recommend using the existing database
and queue manager services on premises or managed services in the cloud,
leaving Kubernetes to do what it is best for, that is, running stateless
containers.
- ***Lockdown***
    - For security reasons, things like actuator endpoints and log levels need to be constrained in a production environment.
    - Externally exposed endpoints should also be reviewed from a security perspective. For example, access to the configuration server should most probably be locked down in a production environment, but we will keep it exposed in this book for convenience.
    - Docker image tags must be specified to be able to track which versions of the microservices have been deployed.
- ***Scale up available resources***
    - To meet the requirements of both high availability
and higher load, we need to run at least two pods per deployment. We might 
also need to increase the amount of memory and CPU that are allowed to be used
per pod. To avoid running out of memory in the Minikube instance, we will keep
one pod per deployment but increase the maximum memory allowed in the
production environment. 
- ***Set up a production-ready Kubernetes cluster***
     - This is outside the scope of this
book, but, if feasible, I recommend using one of the managed Kubernetes services
provided by the leading cloud providers. For the scope of this book, we will
deploy to our local Minikube instance.
## Changes in the source code
The following changes have been applied to the source code to prepare for deployment in
an environment that's used for production:

- A Spring profile named prod has been added to the configuration files in the ***config-repo*** configuration repository:
        ```
        spring.profiles: prod
        ```
- In the ***prod*** profiles, the following has been added:
    - URLs to the resource managers that run as plain Docker containers:
        ```
        spring.rabbitmq.host: 172.17.0.1
        spring.data.mongodb.host: 172.17.0.1
        spring.datasource.url: jdbc:mysql://172.17.0.1:3306/reviewdb
        ```
        We are using the ***172.17.0.1*** IP address to address the Docker engine in
        the Minikube instance. This is the default IP address for the Docker
        engine when creating it with Minikube, at least for Minikube up to
        version 1.2.
        
        There is work ongoing for establishing a standard DNS name for containers to use if they need to access the Docker host they are running on, but at the time of writing this chapter, this work effort hasn't been completed.
        - Log levels have been set to warning or higher, that is, error or fatal. For example:
        ```
        logging.level.root: WARN
        ```
        - The only ***actuator*** endpoints that are exposed over HTTP are the ***info*** and ***health*** endpoints that are used by the liveness 
                and readiness probes in Kubernetes, as well as the ***circuitbreakerevents*** endpoint that's used by the test script, ***test-em-all.bash***:
        ```
        management.endpoints.web.exposure.include: health,info,circuitbreakerevents
        ```
- In the production ***overlay*** folder, ***kubernetes/services/overlays/prod***, one deployment object for each microservice has been added with the following content so that it can be merged with the base definition:
    - For all microservices, v1 is specified as the Docker image tag, and the ***prod*** profile is added to the active Spring profiles. For example (Chapter16/kubernetes/services/overlays/prod/product-prod.yml), we have the following for the ***product*** service:
```Chapter16/kubernetes/services/overlays/prod/product-prod.yml
    spec:
      containers:
      - name: pro
        image: hands-on/product-service:v1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "docker,prod"
```
    - For the Zipkin and configuration server, which don't keep their
configuration in the configuration repository, environment variables
have been added in their deployment definitions with the
corresponding configuration (Chapter16/kubernetes/services/overlays/prod/zipkin-server-prod.yml):
```Chapter16/kubernetes/services/overlays/prod/zipkin-server-prod.yml
    spec:
      containers:
      - name: zipkin-server
        env:
          - name: LOGGING_LEVEL_ROOT
            value: WARN
          - name: RABBIT_ADDRESSES
            value: 172.17.0.1
```
    - Finally, a kustomization.yml (Chapter16/kubernetes/services/overlays/prod/kustomization.yml) file defines that the files in the prodverlay folder shall be merged by specifying the patchesStrategicMerge patch mechanism with the corresponding definition in the base folder:
```Chapter16/kubernetes/services/overlays/prod/kustomization.yml
bases:
- ../../base
patchesStrategicMerge:
- auth-server-prod.yml
- config-server-prod.yml
- gateway-prod.yml
- product-composite-prod.yml
- product-prod.yml
- recommendation-prod.yml
- review-prod.yml
- zipkin-server-prod.yml
```
In a real-world production environment, we should have also changed the
***imagePullPolicy: Never*** setting to ***IfNotPresent***, that is, to
download Docker images from a Docker registry. But since we will be
deploying the production setup to the Minikube instance where we
manually build and tag the Docker images, we will not update this
setting.
## Deploying to Kubernetes
run outside of Kubernetes using Docker Compose. We start them up as we did in the
previous chapters:
```
eval $(minikube docker-env)
docker-compose up -d mongodb mysql rabbitmq
```
We also need to tag the existing Docker images with ***v1*** using the following commands:
```
docker tag hands-on/auth-server hands-on/auth-server:v1
docker tag hands-on/config-server hands-on/config-server:v1
docker tag hands-on/gateway hands-on/gateway:v1
docker tag hands-on/product-composite-service hands-on/product-composite-service:v1
docker tag hands-on/product-service hands-on/product-service:v1
docker tag hands-on/recommendation-service hands-on/recommendation-service:v1
docker tag hands-on/review-service hands-on/review-service:v1
```
From here, the commands are very similar to how we deployed to the development
environment.

We will use another Kustomize overlay and use different credentials for the configuration
server, but, otherwise, it will be the same (which, of course, is a good thing!). We will use
the same configuration repository but configure the pods to use the ***prod*** Spring profile, as
described previously. Follow these steps to do so:

1. Create a namespace, hands-on, and set this as the default namespace
for kubectl:
```
kubectl create namespace hands-on
kubectl config set-context $(kubectl config current-context) --namespace=hands-on
```
2. Create the config map for the configuration repository based on the files in
the config-repo folder with the following command:
```
kubectl create configmap config-repo --from-file=config-repo/ --save-config
```
3. Create the secret for the configuration server with the following command:
```
kubectl create secret generic config-server-secrets \
--from-literal=ENCRYPT_KEY=my-very-secure-encrypt-key \
--from-literal=SPRING_SECURITY_USER_NAME=prod-usr \
--from-literal=SPRING_SECURITY_USER_PASSWORD=prod-pwd \
--save-config
```
4. Create the secret for the clients of the configuration server with the following
command:
```
kubectl create secret generic config-client-credentials \
--from-literal=CONFIG_SERVER_USR=prod-usr \
--from-literal=CONFIG_SERVER_PWD=prod-pwd --save-config
```
5. Remove the clear text encryption key and passwords from the command history:
```
history -c; history -w
```
6. Deploy the microservices for the development environment, based on
the prod overlay, using the -k switch to activate Kustomize, as described
previously:
```
kubectl apply -k kubernetes/services/overlays/prod
```
7. Wait for the deployments to be up and running:
```
kubectl wait --timeout=600s --for=condition=ready pod --all
```
8. To see the Docker images that are currently being used for production, run the
following command:
```
kubectl get pods -o json | jq .items[].spec.containers[].image
```
The response should look something like the following:
```
```
Note the ***v1*** version of the Docker images!

Also note that the resource manager pods for MySQL, MongoDB, and RabbitMQ are gone;
these can be found with the ***docker-compose ps*** command.

Run the test script, ***thest-em-all.bash***, to verify the simulated production environment:
```
HOST=$(minikube ip) PORT=31443 ./test-em-all.bash
```
Expect the same type of output that we got when the test script was run against the
development environment.
