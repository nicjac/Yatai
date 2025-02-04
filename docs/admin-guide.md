# Yatai Administrator's Guide

This guide helps you to install and configure Yatai on a Kubernetes Cluster for your machine
learning team, using the official [Yatai Helm chart](https://github.com/bentoml/yatai-chart). Note
that Helm chart is the official supported method of installing Yatai.

By default, Yatai helm chart will install Yatai and its dependency services in the target Kubernetes cluster. Those dependency services include PostgreSQL, Minio, Docker registry, and Nginx Ingress Controller. Users can configure those services with existing infrastructure or cloud-based services via the Helm chart configuration yaml file.


- [System Overview](#system-overview)
- [Local Minikube Installation](#local-minikube-installation)
- [Production Installation](#production-installation)
  - Custom PostgreSQL database
    - [AWS RDS](#aws-rds)
  - Custom Docker Registry
    - [Docker hub](#docker-hub)
    - [AWR ECR](#aws-ecr)
  - Custom Blob Storage
    - [AWS S3](#aws-s3)
- [Verify Installation](#verify-installation)

## System Overview

### Namespaces:

When deploying Yatai with Helm,  `yatai-system`, `yatai-components`,  `yatai-operators`, `yatai-builders`, and `yatai` in the Kubernetes cluster.

* **yatai-system:**

    All the services that run the yatai platform are group under `yatai-system` namespace.  These services include Yatai application and the default PostgreSQL database.

* **yatai-components:**

    Yatai groups dependency services into components for easy management. These dependency services are deployed in the `yatai-components` namespace. Yatai installs the default deployment component after the service start.

    *Components and their managed services:*

    * *Deployment*: Minio, Docker Registry

    * *Logging*: Loki, Grafana

    * *Monitoring*: Prometheus, Grafana


* **yatai-operators:**

    Yatai uses controllers to manage the lifecycle of Yatai component services. The controllers deployed in the `yatai-operators` namespace.

* **yatai-builders:**

    All the automated jobs such as build docker images for models and bentos are executed in the `yatai-builder` namespace.

* **yatai:**

    By default Yatai server will create a `yatai` namespace on the Kuberentes cluster for managing all the user created bento deployments. User can configure this namespace value in the Yatai web UI.


### Prerequisites:

* *Ingress controller*:

    Yatai uses ingress controller to facilitates access to deployments and canary deployments.


### Default dependency services installed:

* *PostgreSQL*:

    Yatai uses Postgres database to store model and bento’s metadata , deployment configuration, user activities and other information. Users can use their existing database or cloud provider’s services such as AWS RDS.  By default, Yatai will create a Postgres service in the Kubernetes cluster. For production usage, we recommend users to setup external Postgres database from cloud provider such as AWS RDS. This setup provides reliability, high performance and reliability, while persist the data, in case the Kuberentes cluster goes down.

* *Minio datastore*:

    Yatai uses the datastore as persistence layer for storing bentos. By default Yatai will start a Minio service. Users can configure to use cloud-based object store such as AWS S3. Cloud based object stores provide scalability, high performance and reliability at a desirable cost.  They are recommended for production usage.

* *Docker registry*:

    Yatai uses an internal docker registry to provide docker images access for deployments. For users who want to access the built images for other system, they can configure to use DockerHub or other cloud based docker registry services.

See all available helm chart configuration options [here](./helm-configuration.md)

## Local Minikube Installation

Minikube is recommended for development and testing purpose only.

**prerequisites**
- Minikube version 1.20 or newer. Please follow the [official installation guide](https://minikube.sigs.k8s.io/docs/start/) to install Minikube.
- Recommend system with 4 CPUs and 4GB of RAM or more


**Step 1. Start a new minikube cluster**

If you have an existing minikube cluster, make sure to delete it first: `minikube delete`

```bash
# Start a new minikube cluster
minikube start --cpus 4 --memory 4096
```

**Step 2. Enable ingress controller**

```bash
minikube addons enable ingress
```

**Step 3. Add and update Yatai helm chart**

```bash
helm repo add yatai https://bentoml.github.io/yatai-chart
helm repo update
```

**Step 4. Install Yatai chart**

This will create a new namespace `yatai-system` in the Minikube cluster, install Yatai and all its dependency services.

```bash
helm install yatai yatai/yatai -n yatai-system --create-namespace
```

**Step 5. Verify installation**

Go to [Verify installation](#verify-installation) section


## Production Installation

To install and operate Yatai in production, we generally recommend using a dedicated database service(e.g. AWS RDS) and a managed blob storage (AWS S3 or managed MinIO cluster). The following demonstrates how to customize a Yatai deployment for production use.

**Prerequisite**

- Kubernetes cluster with version 1.20 or newer
- LoadBalancer (If you are using AWS EKS, Google GKS, your cluster is likely already configured with a working LoadBalancer. If you are using Kubernetes in a private data center, contact your system admin)

**Install Yatai with default configuration.**

1. Download and update Helm repo

    ```bash
    helm repo add yatai https://bentoml.github.io/yatai-chart

    helm repo update
    
    ```
    Create yatai-system namespace
    ```bash
    kubectl create namespace yatai-system
    ```

2. Install Yatai helm chart

    ```bash
    helm install yatai yatai/yatai -n yatai-system
    ```

### Custom PostgreSQL database

#### AWS RDS

Prerequisites:

- `jq`  command line tool. Follow the [official installation guide](https://stedolan.github.io/jq/download/) to install `jq`
- AWS CLI with RDS permission. Follow the [official installation guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install AWS CLI

1. Create an RDS db instance and secret

    ```bash
    USER_PASSWORD=secret99
    USER_NAME=admin
    DB_NAME=yatai
    INSTANCE_IDENTIFIER=yatai-postgres

    aws rds create-db-instance \
        --db-name $DB_NAME \
        --db-instance-identifier $INSTANCE_IDENTIFIER \
        --db-instance-class db.t3.micro \
        --engine postgres \
        --master-username $USER_NAME \
        --master-user-password $USER_PASSWORD \
        --allocated-storage 20
    
    kubectl create secret generic rds-password \
        --from-literal=password=$USER_PASSWORD \
        -n yatai-system
    ```
    Make sure your RDS db allows has a security group that will allow Yatai to access it.

2. Get the RDS instance’s endpoint and port information

    ```bash
    read ENDPOINT PORT < <(echo $(aws rds describe-db-instances --db-instance-identifier $INSTANCE_IDENTIFIER | jq '.DBInstances[0].Endpoint.Address, .DBInstances[0].Endpoint.Port'))
    ```


1. Install Yatai chart with the RDS configuration

    ```bash
    DB_NAME=yatai

    helm install yatai yatai/yatai \
    	--set postgresql.enabled=false \
    	--set externalPostgresql.host=$host \
    	--set externalPostgresql.port=$port \
    	--set externalPostgresql.user=$USER_NAME \
    	--set externalPostgresql.existingSecret=rds-password \
        --set externalPostgresql.existingSecretPasswordKey=password \
    	--set externalPostgresql.database=$DB_NAME \
    	-n yatai-system
    ```


### Custom Docker registry

#### Docker hub

```bash
helm install yatai yatai/yatai \
	--set config.docker_registry.server='https://index.docker.io/v1' \
	--set config.docker_registry.username='MY_DOCKER_USER' \
	--set config.docker_registry.password='MY_DOCKER_USER_PASSWORD' \
	--set config.docker_registry.secure=true \
	-n yatai-system
```

#### AWS ECR

Prerequisites:

- `jq`  command line tool. Follow the [official installation guide](https://stedolan.github.io/jq/download/) to install `jq`
- AWS CLI with AWS ECR permission. Follow the [official installation guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install AWS CLI


1. Create AWS ECR repositories:
    1. Create repositories using default registry ID:

        ```bash
        BENTO_REPO=yatai-bentos
        MODEL_REPO=yatai-models

        aws ecr create-repository --repository-name $BENTO_REPO
        aws ecr create-repository --repository-name $MODEL_REPO

        # If the repositories are created using the default registry
        read ENDPOINT < <(echo $(aws ecr get-authorization-token | jq '.authorizationData[0].proxyEndpoint'))
        ```

    2. Create repositories using a specific registry ID for Yatai:

        ```bash
        BENTO_REPO=yatai-bentos
        MODEL_REPO=yatai-models
        REGISTRY_ID=yatai

        aws ecr create-repository --registry-id $REGISTRY_ID --repository-name $BENTO_REPO
        aws ecr create-repository --registry-id $REGISTRY_ID --repository-name $MODEL_REPO

        # If the repositories are created use a different registry id from the default
        read ENDPOINT < <(echo $(aws ecr get-authorization-token --regsitry-ids $REGISTRY_ID | jq '.authorizationData[0].proxyEndpoint'))
        ```

2. Get ECR registry password info

    ```bash
    PASSWORD=$(aws ecr get-login-password)
    ```

3. Create Kubernetes secrets

    ```bash
    kubectl create secret generic yatai-docker-registry-credentials \
        --from-literal=password=$PASSWORD
        -n yatai-system
    ```

3. Install Yatai chart

    ```bash
    helm install yatai yatai/yatai \
    	--set externalDockerRegistry.enabled=true \
    	--set externalDockerRegistry.server=$ENDPOINT \
    	--set externalDockerRegistry.username=AWS \
    	--set externalDockerRegistry.secure=true \
    	--set externalDockerRegistry.bentoRepositoryName=$BENTO_REPO \
    	--set externalDockerRegistry.modelRepositoryName=$MODEL_REPO \
    	--set externalDockerRegistry.existingSecret=yatai-docker-registry-credentials \
    	--set externalDockerRegistry.existingSecretPasswordKey=password \
    	-n yatai-system
    ```


### Custom blob storage

#### AWS S3

Prerequisites:

- AWS CLI with AWS S3 permission. Follow the [official installation guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to install AWS CLI


1. Configure AWS S3 bucket for Yatai

    ```bash
    BUCKET_NAME=my_yatai_bucket
    MY_REGION=us-west-1
    ENDPOINT='s3.amazonaws.com'

    aws s3 create-bucket --bucket $BUCKET_NAME --region MY_REGION
    ```

2. Create Kubernetes secrets

    ```bash
    AWS_ACCESS_KEY_ID=$(aws configure get default.aws_access_key_id)
    AWS_SECRET_ACCESS_KEY=$(aws configure get default.aws_secret_access_key)
    kubectl create secret generic yatai-s3-credentials \
        --from-literal=accessKeyId=$AWS_ACCESS_KEY_ID \
        --from-literal=secretAccessKey=$AWS_SECRET_ACCESS_KEY
        -n yatai-system
    ```

3. Install Yatai chart

    ```bash
    helm install yatai yatai/yatai \
        --set externalS3.enabled=true \
        --set externalS3.endpoint=$ENDPOINT \
    	--set externalS3.region=$MY_REGION \
    	--set externalS3.bucketName=$BUCKET_NAME \
    	--set externalS3.secure=true \
        --set externalS3.existingSecret="yatai-s3-credentials" \
    	--set externalS3.existingSecretAccessKeyKey=accessKeyId \
    	--set externalS3.existingSecretSecretKeyKey=secretAccessKey \
    	-n yatai-system
    ```

### Verify Installation

> NOTE: If you don't have `kubectl` installed and you use `minikube`, you can use `minikube kubectl --` instead of `kubectl`, for more details on using it, please check: [minikube kubectl](https://minikube.sigs.k8s.io/docs/commands/kubectl/)

#### 1. Check helm release installation status with following command:

```bash
helm status yatai -n yatai-system
```

The output should be:

```bash
NAME: yatai
LAST DEPLOYED: Wed Jul 27 17:53:07 2022
NAMESPACE: yatai-system
STATUS: deployed
REVISION: 1
NOTES:
When installing Yatai for the first time, run the following command to get an initialzation link for creating your admin account:

  export YATAI_INITIALIZATION_TOKEN=$(kubectl get secret yatai --namespace yatai-system -o jsonpath="{.data.initialization_token}" | base64 --decode)

  echo "Create admin account at: http://127.0.0.1:8080/setup?token=$YATAI_INITIALIZATION_TOKEN" && kubectl --namespace yatai-system port-forward svc/yatai 8080:80
```

#### 2. Check the yatai-system status

> NOTE: Since yatai needs to wait for the postgresql pod to start successfully first, so the pod of yatai will restart several times, please be patient and wait for a while, the pod of yatai will start successfully in about 3 minutes.

Check the yatai-system pod status:

```bash
kubectl -n yatai-system get pod
```

The output should be:

```bash
NAME                     READY   STATUS    RESTARTS       AGE
yatai-7cf7c4f7f7-wkhjn   1/1     Running   4 (2m1s ago)   4m7s
yatai-postgresql-0       1/1     Running   0              4m7s
```

#### 3. Check the yatai-operators status

Check the yatai-operators pod status:

```bash
kubectl -n yatai-operators get pod
```

The output should be:

```bash
NAME                                                         READY   STATUS    RESTARTS   AGE
deployment-yatai-deployment-comp-operator-665dd86559-rkg4l   1/1     Running   0          5m3s
```

Verify that yatai-deployment-comp-operator is running properly by checking the logs:

```bash
kubectl -n yatai-operators logs -f deploy/deployment-yatai-deployment-comp-operator
```

The output should be:

```bash
time="2022-07-27T18:00:04Z" level=info msg="Deployment is not ready: yatai-components/yatai-docker-registry. 0 out of 1 expected pods are ready"
1.658944806495706e+09   INFO    controller.deployment   helm release yatai-docker-registry installed successfully       {"reconciler group": "component.yatai.ai", "reconciler kind": "Deploymen
t", "name": "deployment", "namespace": ""}
1.6589448065974028e+09  INFO    controller.deployment   creating daemonset docker-private-registry-proxy ...    {"reconciler group": "component.yatai.ai", "reconciler kind": "Deployment", "nam
e": "deployment", "namespace": ""}
1.6589448066120095e+09  INFO    controller.deployment   Installed DockerRegistryComponent successfully  {"reconciler group": "component.yatai.ai", "reconciler kind": "Deployment", "name": "dep
loyment", "namespace": ""}
1.6589448066232717e+09  INFO    controller.deployment   Congratulation! All components are installed!   {"reconciler group": "component.yatai.ai", "reconciler kind": "Deployment", "name": "dep
loyment", "namespace": ""}

```

#### 4. Check the yatai-components status

Check pod status:

```bash
kubectl -n yatai-components get pod
```

The output should be:

```bash
NAME                                               READY   STATUS    RESTARTS   AGE
cert-manager-868bc96c6d-z56sg                      1/1     Running   0          9m21s
cert-manager-cainjector-9cc6bbc8b-8lqc9            1/1     Running   0          9m21s
cert-manager-webhook-77965c59b5-wbscj              1/1     Running   0          9m21s
docker-private-registry-proxy-skb8b                1/1     Running   0          5m34s
minio-operator-765bcdf9c4-nnvh2                    1/1     Running   0          7m56s
yatai-csi-driver-image-populator-prskt             2/2     Running   0          8m
yatai-docker-registry-7b8f6b4f59-ckkxr             1/1     Running   0          6m40s
yatai-minio-console-7d568cc8bc-xgffd               1/1     Running   0          7m56s
yatai-minio-ss-0-0                                 1/1     Running   0          7m1s
yatai-minio-ss-0-1                                 1/1     Running   0          7m1s
yatai-minio-ss-0-2                                 1/1     Running   0          7m1s
yatai-minio-ss-0-3                                 1/1     Running   0          7m
yatai-yatai-deployment-operator-8476ff78b5-jsgnw   1/1     Running   0          8m20s
```

#### 5. Setup

You can access the Yatai Web UI: http://${Yatai URL}/setup?token=<token>. You can find the Yatai URL link and the token again using `helm get notes yatai -n yatai-system` command.

You can also retrieve the token using `kubectl` command:

```bash
export YATAI_INITIALIZATION_TOKEN=$(kubectl get secret yatai --namespace yatai-system -o jsonpath="{.data.initialization_token}" | base64 --decode)
```

