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

## Deploying to Kubernetes
