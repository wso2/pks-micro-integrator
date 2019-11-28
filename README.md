# pks-micro-integrator

This repository contains Micro Integrator Pivotal PKS artifact resources.

# Prerequisites

* An already setup [Pivotal Application service(PAS) (v2.7.1)](https://network.pivotal.io/products/elastic-runtime#/releases/477769)
* An already setup [Pivotal Container Service(PKS) (v1.5.1)](https://network.pivotal.io/products/pivotal-container-service#/releases/481414)
* Install [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/) version [nginx-0.22.0](https://github.com/kubernetes/ingress-nginx/releases/tag/nginx-0.22.0) on the PKS provisioned Kubernetes cluster
* An already setup [Pivotal Container Service Manager(KSM) (v0.5.1)](https://network.pivotal.io/products/container-services-manager#/releases/493270) 
* An already setup [VMware Harbor Registry (v1.8.3)](https://network.pivotal.io/products/harbor-container-registry#/releases/470132) 

>In the context of this document,

> **PKS_MICRO_INTEGRATOR_HOME** will refer to a local copy of the wso2/pks-micro-integrator Git repository.

# Build the PKS compatible Micro Integrator Helm chart

1. Clone PKS Resources for WSO2 Micro Integrator Git repository.

    `https://github.com/wso2/pks-micro-integrator.git`

1. Goto PKS_MICRO_INTEGRATOR_HOME/helm/micro-integrator directory.

    `cd PKS_MICRO_INTEGRATOR_HOME/helm/micro-integrator`

1. Build the Micro Integrator Helm chart with the following command
   
    ```helm package .```

# Configuring Micro Integrator on PKS

1. Push docker images to the private registry	
    ```
    docker pull wso2/micro-integrator:1.1.0
    docker tag wso2/micro-integrator:1.1.0  PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/micro-integrator:1.1.0
    docker push PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/micro-integrator:1.1.0
    
    docker pull busybox:1.31
    docker tag busybox:1.31 PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/busybox:1.31
    docker push PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/busybox:1.31 
    ```
   Where:
    * **PRIVATE_DOCKER_REGISTRY_DOMAIN** refers to the docker registry domain used by PKS.
    * **PROJECT** refers to the Harbor private docker registry project that is configured for the Pivotal KSM(Container Services Manager for Pivotal Platform -> Settings -> Container Registry Configuration -> Registry Server URL).

1.
    1. Setup Micro Integrator Service on Pivotal KSM
        1. Install the [KSM CLI](https://network.pivotal.io/products/container-services-manager#/releases/493270/file_groups/2225)
        1. Configure KSM CLI using the [Pivotal KSM documentation](https://docs.pivotal.io/ksm/0-5/using.html)
        1. Enable Service Offering Access
            1. Log in to your PAS deployment by running the following command:
            
                `cf login -a API-URL -u USERNAME -p PASSWORD`
           
               Where:
                * **API-URL** is the API endpoint for your PAS instance.
                * **USERNAME** is your username for your PAS instance.
                * **PASSWORD** is your password for your PAS instance.

            1. Enable access to the Micro Integrator service for all organizations by running the following command:
                
                `cf enable-service-access micro-integrator`
            1. View the newly created Micro Integrator service plans by running the following command:
                
                `cf marketplace`
                    
    Please find more information on using KSM [here](https://docs.pivotal.io/ksm/0-5/using.html).

# Micro Integrator service plans

Micro Integrator service comes with two plans which represent the resources allocated for a Micro integrator service instance:
* Small [CPU = 1000m, Memory = 1Gi]
* Medium [CPU = 2000m, Memory = 2Gi]

# Deploy Micro Integrator service instance

### Burning Carbon Application(CApp) into Micro Integrator base image

In this approach, the CApp is burnt into the Micro integrator docker image .Thereafter the image is referenced when deploying the Micro integrator service instance

1. Creating the Micro Integrator Docker image
    1. Create a file named Dockerfile with the following content 
        ```
           FROM wso2/micro-integrator:1.1.0
           ARG CAPP_LOCATION
           COPY  ${CAPP_LOCATION} /home/wso2carbon/wso2mi/repository/deployment/server/carbonapps
        ```
    
    1. Copy the CApp to the same directory where the Dockerfile is located.
    1. Build the Docker image with the following command:
    
        `docker build -t PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME:1.1.0 --build-arg CAPP_LOCATION=./CAPP_FILE .`
        
        Where:
         * **PRIVATE_DOCKER_REGISTRY_DOMAIN** refers to the docker registry domain used by PKS.
         * **PROJECT** refers to the Harbor private docker registry project that is configured for the Pivotal KSM(Container Services Manager for Pivotal Platform -> Settings -> Container Registry Configuration -> Registry Server URL).
         * **IMAGE_NAME** is the name of the docker image.
         * **CAPP_FILE** is the CApp file name(Ex: HelloWorld.car).
     
    1. Push the docker image to the private registry with the following command:
    
        `docker push PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME:1.1.0 `

1. Create a file(params.json) which contains parameters for Micro integrator service instance 
    ```
    {
    "images": {
     "microIntegrator": {
       "image": "PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME"
     }
    },
     "ingress": {
       "host": "HOST_NAME"
     }
    }
    ```
    Where:
     * **PRIVATE_DOCKER_REGISTRY_DOMAIN** refers to the docker registry domain used by PKS.
     * **PROJECT** refers to the Harbor private docker registry project that is configured for the Pivotal KSM(Container Services Manager for Pivotal Platform -> Settings -> Container Registry Configuration -> Registry Server URL).
     * **IMAGE_NAME** is the name of the docker image.
     * **HOST_NAME** is the Ingress hostname for Micro integrator service instance. 
     
1. Create Micro Integrator Service Instance by running the following command:
    ```
    cf create-service micro-integrator PLAN SERVICE_INSTANCE_NAME -c params.json
    ```
    Where:
    
      * **PLAN** is the Micro integrator service plan(small or medium).
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **params.json** is a file that contains parameters for the Micro integrator service instance. 

1. Create a service key for the Micro-integrator by running the following command:

    `cf create-service-key SERVICE_INSTANCE_NAME  KEY_NAME`
    
    Where:
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **KEY_NAME** is the service key name.

1. Get the service key content with the following command:

   `cf service-key SERVICE_INSTANCE_NAME  KEY_NAME`
   
    Where:
     * **SERVICE_INSTANCE_NAME** is the service instance name.
     * **KEY_NAME** is the service key name.
    
    Example output:
    ```      
    {
     "hostname": "HOSTNAME",
     "namespace": "NAMESPACE",
     "releaseName": "RELEASE_NAME"
    }
    ```

1. Configure a hostname mapping to point **HOSTNAME** to Ingress controller Loadbalancer.

1. Invoke the Micro Integrator service instance as shown below

    `curl https://HOSTNAME/servies/SERVICE`

### Hosting CApp in a remote location

In this method, the CApp is hosted in a remote location and the location is passed as a parameter when deploying the Micro integrator service instance.

1. Create a file(params.json) which contains parameters for Micro integrator service instance 
    ```
    {
     "capp": {
       "urls": "CAPP_FILE_LOCATION"
     },
     "ingress": {
       "host": "HOST_NAME"
     }
    }
    ```
    Where:
     * **CAPP_FILE_LOCATION**  is the CApp file remote location(Ex: https://download.com/helloworld.capp).
     * **HOST_NAME** is the Ingress hostname for Micro integrator service instance. 
     
1. Create Micro Integrator Service Instance by running the following command:
    ```
    cf create-service micro-integrator PLAN SERVICE_INSTANCE_NAME -c params.json
    ```
    Where:
    
      * **PLAN** is the Micro integrator service plan(small or medium).
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **params.json** is a file that contains parameters for the Micro integrator service instance. 

1. Create a service key for the Micro-integrator by running the following command:

    `cf create-service-key SERVICE_INSTANCE_NAME  KEY_NAME`
    
    Where:
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **KEY_NAME** is the service key name.

1. Get the service key content with the following command:

   `cf service-key SERVICE_INSTANCE_NAME  KEY_NAME`
   
    Where:
     * **SERVICE_INSTANCE_NAME** is the service instance name.
     * **KEY_NAME** is the service key name.
    
    Example output:
    ```aidl      
    {
     "hostname": "HOSTNAME",
     "namespace": "NAMESPACE",
     "releaseName": "RELEASE_NAME"
    }
    ```

1. Configure a hostname mapping to point **HOSTNAME** to Ingress controller Loadbalancer.

1. Invoke the Micro Integrator service instance as shown below

    `curl https://HOSTNAME/servies/SERVICE`

# Override Micro Integrator default configurations

The configuration file of the Micro Integrator is located at MICRO_INTEGRATOR_HOME/conf/deployment.tom.l In the above
 location  MICRO_INTEGRATOR_HOME  refers to unzipped Micro Integrator distribution. There are two ways to provide this
  configuration file to a Micro integrator service instance.


### Burning configuration file into Micro Integrator base image

In accordance with this approach, the deployment.toml file is burnt into the Micro integrator docker image itself.
The image is referenced when deploying the Micro integrator service instance.


1. Creating the Micro Integrator Docker image
    1. Create a file named Dockerfile with the following content 
        ```
        FROM wso2/micro-integrator:1.1.0
        ARG CONFIG_LOCATION
        COPY  ${CONFIG_LOCATION} /home/wso2carbon/wso2mi/conf/
        ```
        This step can be combined with the Burning CApp into Micro Integrator base Docker image step.  
    1. Copy the deployment.toml to the same directory where the Dockerfile is located.
    1. Build the Docker image with the following command:
    
        `docker build -t PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME:1.1.0 --build-arg CONFIG_LOCATION=./deployment.toml .`
        
        Where:
         * **PRIVATE_DOCKER_REGISTRY_DOMAIN** refers to the docker registry domain used by PKS.
         * **PROJECT** refers to the Harbor private docker registry project that is configured for the Pivotal KSM(Container Services Manager for Pivotal Platform -> Settings -> Container Registry Configuration -> Registry Server URL).
         * **IMAGE_NAME** is the name of the docker image.
         * **CONFIG_LOCATION** is the deployment.toml file path.
     
    1. Push the docker image to the private registry with the following command:
    
        `docker push PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME:1.1.0 `

1. Create a file(params.json) which contains parameters for Micro integrator service instance 
    ```
    {
    "images": {
     "microIntegrator": {
       "image": "PRIVATE_DOCKER_REGISTRY_DOMAIN/PROJECT/IMAGE_NAME"
     }
    },
     "ingress": {
       "host": "HOST_NAME"
     }
    }
    ```
    Where:
     * **PRIVATE_DOCKER_REGISTRY_DOMAIN** refers to the docker registry domain used by PKS.
     * **PROJECT** refers to the Harbor private docker registry project that is configured for the Pivotal KSM(Container Services Manager for Pivotal Platform -> Settings -> Container Registry Configuration -> Registry Server URL).
     * **IMAGE_NAME** is the name of the docker image.
     * **HOST_NAME** is the Ingress hostname for Micro integrator service instance. 
     
1. Create Micro Integrator Service Instance by running the following command:
    ```
    cf create-service micro-integrator PLAN SERVICE_INSTANCE_NAME -c params.json
    ```
    Where:
    
      * **PLAN** is the Micro integrator service plan(small or medium).
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **params.json** is a file that contains parameters for the Micro integrator service instance. 

1. Create a service key for the Micro-integrator by running the following command:

    `cf create-service-key SERVICE_INSTANCE_NAME  KEY_NAME`
    
    Where:
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **KEY_NAME** is the service key name.

1. Get the service key content with the following command:

   `cf service-key SERVICE_INSTANCE_NAME  KEY_NAME`
   
    Where:
     * **SERVICE_INSTANCE_NAME** is the service instance name.
     * **KEY_NAME** is the service key name.
    
    Example output:
    ```    
    {
     "hostname": "HOSTNAME",
     "namespace": "NAMESPACE",
     "releaseName": "RELEASE_NAME"
    }
    ```

1. Configure a hostname mapping to point **HOSTNAME** to Ingress controller Loadbalancer.

1. Invoke the Micro Integrator service instance as shown below

    `curl https://HOSTNAME/servies/SERVICE`

### Hosting the Configuration file in a remote location

In this approach, the deployment.toml is hosted in a remote location and the location is passed as a parameter when deploying the Micro integrator service instance.

1. Create a file(params.json) which contains parameters for Micro integrator service instance 
    ```
    {
     "ingress": {
       "host": "HOST_NAME"
     },
     "deploymentTomlURL": "CONFIG_FILE_REMOTE_LOCATION"
    }
    ```
    Where:
     * **HOST_NAME** is the Ingress hostname for Micro integrator service instance. 
     * **CONFIG_FILE_REMOTE_LOCATION**openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key -out cert -subj "/CN=*.maanadev.com/O=wso2"openssl x509 -in cert -text -noout is the deployment.toml file remote location(Ex: https://download.com/deplyment.toml).
     
1. Create Micro Integrator Service Instance by running the following command:
    ```
    cf create-service micro-integrator PLAN SERVICE_INSTANCE_NAME -c params.json
    ```
    Where:
    
      * **PLAN** is the Micro integrator service plan(small or medium).
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **params.json** is a file that contains parameters for the Micro integrator service instance. 

1. Create a service key for the Micro-integrator by running the following command:

    `cf create-service-key SERVICE_INSTANCE_NAME  KEY_NAME`
    
    Where:
      * **SERVICE_INSTANCE_NAME** is the service instance name.
      * **KEY_NAME** is the service key name.

1. Get the service key content with the following command:

   `cf service-key SERVICE_INSTANCE_NAME  KEY_NAME`
   
    Where:
     * **SERVICE_INSTANCE_NAME** is the service instance name.
     * **KEY_NAME** is the service key name.
    
    Example output:
    ```      
    {
     "hostname": "HOSTNAME",
     "namespace": "NAMESPACE",
     "releaseName": "RELEASE_NAME"
    }
    ```

1. Configure a hostname mapping to point **HOSTNAME** to Ingress controller Loadbalancer.

1. Invoke the Micro Integrator service instance as shown below

    `curl https://HOSTNAME/servies/SERVICE`
