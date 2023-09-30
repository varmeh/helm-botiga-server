# Helm Chart for Botiga Backend

- [Helm Chart for Botiga Backend](#helm-chart-for-botiga-backend)
  - [Debug Templates](#debug-templates)
  - [Installation](#installation)
    - [1. Ingress Controller](#1-ingress-controller)
    - [2. Cert-Manager](#2-cert-manager)
    - [3. App Installation](#3-app-installation)
      - [Image Architecture](#image-architecture)
      - [Create Namespace](#create-namespace)
      - [Set Default Namespace](#set-default-namespace)
      - [Create Secrets](#create-secrets)
        - [Docker Registry Secret](#docker-registry-secret)
        - [App Secrets](#app-secrets)
        - [Verifying Secrets](#verifying-secrets)
        - [Mounting Firebase SDK File](#mounting-firebase-sdk-file)
      - [Install App](#install-app)
        - [Local Installation](#local-installation)
        - [Cloud Installation](#cloud-installation)
  - [Uninstall Chart](#uninstall-chart)
  - [Debug Installation](#debug-installation)

## Debug Templates

- To debug the templates, you can use the following command:

```bash
helm template prod -f value.prod.yaml . --debug > tmp/templates.yaml
```

- This will dump the template manifests into a file called `templates.yaml` in the current directory
- You can then use this file to debug the templates

## Installation

- Installation on the Cloud Provider is a 3 part process:
  1. [Ingress Controller](#1-ingress-controller) - A Cluster Wide Ingress Controller for all namespaces
  2. [Cert-Manager](#2-cert-manager) - A Cluster Wide Cert-Manager for all namespaces
  3. [App Installation](#3-app-installation) - Installation of the App in a specific namespace
- 1st 2 steps are one time setup & could be skipped if already done
- 1st 2 steps are also `OPTIONAL` for testing on a local machine

---

### 1. Ingress Controller

- `Ingress Controller` is a combination of resources that routes traffic from clound provider to the application
- To understand it better, refer [documentation](https://varmeh.github.io/tech-docs/docs/deployment/k8s/K8sPrimer.html#ingress)

- There are 2 major ingresses available:

| Ingress Name               | Description                                              | URL                                                      | Cost                    |
|----------------------------|----------------------------------------------------------|----------------------------------------------------------|-------------------------|
| Kubernetes/ingress-nginx   | Open Source Ingress managed by K8s Community.            | [Link](https://github.com/kubernetes/ingress-nginx)      | Totally Free            |
| Nginx-Ingress              | From Nginx Inc. Free with limited features.              | [Link](https://docs.nginx.com/nginx-ingress-controller)  | Free (Limited) / Nginx+ |

- Going with `Kubernetes/ingress-nginx`, as it's free & does not require any additional setup
<!-- 
#### Modifications in `values.yaml`

```yaml
deployment:
  service:
    type: ClusterIP
    port: 80

ingress:
  enabled: true
  className: "nginx"
```

- Change `service.type` to `ClusterIP` and remove `nodePort` as it is not required for ingress
- Set `ingress.enabled` to `true` to enable ingress
- Routes to it could be configured in `ingress.hosts` in `values.yaml`
- This will install an `ingress resource` which specifies the routes to the application
- To test it on local machine, you need to add the following entry to your `/etc/hosts` file:

```bash
127.0.0.1 chart.local
```

- Chart.local is your defined `ingress.hosts[0].host` url in values.yaml file
- Post the changes, follow the [commons steps](#common-steps) -->

- `Ingress Controller` would be installated at `Cluster Level` as a `Shared resource`
- This shared ingress controller would manage the `routing for all the namespaces` in the cluster
- All a namespace needs to do is to create an `ingress resource` with the required routes

- To begin, create a namespace for ingress controller:

```bash
kubectl create namespace ingress
```

- Instal ingress controller in this namespace:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install cluster-ingress ingress-nginx/ingress-nginx -n ingress
```

- This will also provision a `External Load Balancer(ELB)` in the cloud provider which will be used to route traffic to the k8s cluster

### 2. Cert-Manager

- Cert-Manager is a Kubernetes add-on to automate the management and issuance of TLS certificates from various issuing sources.
- It is used to manage SSL Certificates for the application
- `SSL Termination` in k8s can be done at 2 levels:
  - `External Load Balancer` - Provisioned by Cloud Provider
  - `Ingress Controller`

Following table explains the pros & cons of both approaches:

| Feature/Aspect                    | External Load Balancer Termination | Ingress Controller Termination  |
|-----------------------------------|------------------------------------|---------------------------------|
| `SSL Processing Overhead`       | Offloaded to Load Balancer        | Handled by Kubernetes nodes     |
| `SSL Certificate Management`    | Managed externally (*manual* or *specific integrations*) | Managed within the cluster with `cert-manager` |
| `Connection Encryption`         | Encrypted only up to the Load Balancer | Encrypted end-to-end up to the pod |
| `Centralized SSL Management`    | Yes (*all certs managed in one place*) | No (*each ingress might have its own certs*) |
| `Cost`                          | Potential extra costs for LB-based SSL processing | Might save on LB costs but use more node resources |
| `Ease of Setup`                 | Varies & Depends on Cloud Provider | Consistent with `cert-manager` |
| `End-to-end Encryption`         | No (traffic decrypted at LB)      | Yes (fully encrypted up to the pod) |
| `Integration with Let's Encrypt`| Manual or specific integrations  | Direct (e.g., `cert-manager`)    |
| `Auto Renewal of Certificates`  | Depends on provider               | Generally automated with `cert-manager` |
| `Latency`                       | Potential reduction as SSL processing is offloaded | Slight increase due to processing within the cluster |

- For this project, we'll go with `SSL Termination at Ingress Controller` following reasons:
  - Fully Automated Setup with `cert-manager`
  - Auto Renewal of Certificates
  - Consistent across all Cloud Providers
  - Integration with `Let's Encrypt`

- `Installation Process`:

- Add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

- Install the Cert-Manager Helm chart:

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.13.1 --set installCRDs=true 
```

- `Docs for Reference`: following docs illustrate the steps for SSL Managament at ELB & Ingress-Controller seperately:
  - [SSL at Digital Ocean External Load Balancer](./docs/Ssl@Elb.md)
  - [SSL Using Ingress Controller](./docs/Ssl@Ingress.md)

### 3. App Installation

- Installation would vary based on underlying infra
- For a `cloud provider`, please ensure the deployment of `Ingress Controller` & `Cert-Manager` is complete
- Use following values.yaml file, based on the infra:
  - `values.yaml` - For local installation (*Docker Desktop*)
  - `values.dev.yaml` - For Development Environment on Cloud Provider
  - `values.prod.yaml` - For Production Environment on Cloud Provider

#### Image Architecture

- The image architecture should match the node architecture
- To check the node architecture, run the following command:

```bash
kubectl get nodes -o=jsonpath='{.items[0].status.nodeInfo.architecture}'
```

- Then, ensure that you have a docker image with the same architecture
- To create an image with a specific architecture, you can use the following command:

```bash
docker buildx create --use
docker buildx build --platform linux/amd64 -t varunbotiga/botiga-server:1.0.0-amd64 . --push
```

- Platform could have multiple values like `linux/amd64,linux/arm64,linux/arm/v7`
- Image Name recommended format is `<docker-username>/<image-name>:<version>-<architecture>`

#### Create Namespace

- As this cluster could be used for multiple applications with different environments, please create a namespace for the application
- Nomenclature the namespace as `<env>-<app-name>`, e.g. `dev-botiga` `prod-botiga`

```bash
kubectl create namespace prod-botiga
```

#### Set Default Namespace

- This steps is optional
- It simply avoids the need of adding `-n prod-botiga` to every `kubectl` command

```bash
kubectl config set-context --current --namespace=prod-botiga
```

#### Create Secrets

##### Docker Registry Secret

- Secret for pulling images from docker registry of type - `docker-registry`

```bash
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=docker.io \
  --docker-username=varunbotiga \
  --docker-password=<Your-Docker-Registry-Token> \
  --docker-email=varun@botiga.app
```

##### App Secrets

- Upload app confidential information from `.env` file to a secret of type - `generic`
- This approach gives us flexibility to set custom secret values based on environments

```bash
kubectl create secret generic app-secret --from-env-file=.env.prod
```

##### Verifying Secrets

- To verify the secrets, you can use the following command:

```bash
kubectl describe secrets app-secret
```

- This will not show the actual secret values, but will give you other metadata like when the secret was created.

- To verify the secret values, you can use the following command:

```bash
kubectl get secret docker-registry-secret -o jsonpath="{.data.\.dockerconfigjson}" | base64 --decode
kubectl get secret app-secret -o jsonpath="{.data.NODE_ENV}" | base64 --decode
```

##### Mounting Firebase SDK File

- Firebase SDK File is required to access Firebase Services from the application
- As it a JSON file, it need to be `mount as a volume` & the path of the volume should be set as our `GOOGLE_APPLICATION_CREDENTIALS` environment variable
- To mount this file, `Create a Secret from JSON File`:
  
```bash
kubectl create secret generic firebase-sdk --from-file=firebase-sdk.json=<path-to-firebase-sdk-json-file>
```

- Please ensure that the variable name in secret is `firebase-sdk.json` as it is referenced in the deployment template

- Accessing in Template:

`Note`: Following 2 code pieces are already embedded in deployment template & are here only for clarity:

1. Mount the Secret as a Volume in a Pod:

    ```yaml
      apiVersion: apps/v1
      kind: Deployment
      spec:
        template:
          spec:
            containers:
            - name: my-nodejs-container
              image: my-nodejs-image
              volumeMounts:
              - name: firebase-sdk-volume
                mountPath: "/etc/firebase-sdk"
            volumes:
            - name: firebase-sdk-volume
              secret:
                secretName: firebase-sdk
    ```

2. Set variable Path to this file:

    ```yaml
      env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: "/etc/firebase-sdk/firebase-sdk.json"
    ```

#### Install App

##### Local Installation

- For local installation `ingress.enabled` is set to `false` in `values.yaml`
- `deployment.service.type` is set to `NodePort` with a `nodeport` value.

```bash
helm install prod . -f values.yaml
```

- If installation is successful and `service.type` is `NodePort`, then, service for Docker-Desktop could be tested at `http://localhost:<node-port>`

##### Cloud Installation

- To install the app, run the following command:

```bash
helm install prod . -f values.prod.yaml
```

---

## Uninstall Chart

```bash
helm uninstall prod
```

- [Helm Cheat Sheet](https://helm.sh/docs/intro/cheatsheet/)

## Debug Installation

- Check all the resouces created:

```bash
kubectl get all
```

- You can also check the status of the pods:

```bash
kubectl get pods
```

- Describe Pod for details:

```bash
kubectl describe pod <pod-name>
```

- Check the logs of the pod:

```bash
kubectl logs <pod-name>
```

- Check 1st 100 lines of logs:

```bash
kubectl logs [POD_NAME] | head -n 100
```

- Access Shell of the Container

```bash
kubectl exec -it <pod_name> -n -- /bin/sh
```
