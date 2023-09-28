# Helm Chart for Botiga Backend

- [Helm Chart for Botiga Backend](#helm-chart-for-botiga-backend)
  - [Debug Templates](#debug-templates)
  - [Installation](#installation)
    - [Common Steps](#common-steps)
      - [Create Namespace](#create-namespace)
      - [Set Default Namespace](#set-default-namespace)
      - [Create Secrets](#create-secrets)
        - [Docker Registry Secret](#docker-registry-secret)
        - [App Secrets](#app-secrets)
        - [Verifying Secrets](#verifying-secrets)
        - [Mounting Firebase SDK File](#mounting-firebase-sdk-file)
    - [Install App with a NodePort Service](#install-app-with-a-nodeport-service)
    - [Install App with Ingress](#install-app-with-ingress)
      - [Modifications in `values.yaml`](#modifications-in-valuesyaml)
      - [Types of Installation](#types-of-installation)
        - [Isolated Ingress Controller](#isolated-ingress-controller)
        - [Shared Ingress Controller](#shared-ingress-controller)
        - [Ingress Controller Overview](#ingress-controller-overview)
    - [Uninstall Chart](#uninstall-chart)
  - [Debug Installation](#debug-installation)

## Debug Templates

- To debug the templates, you can use the following command:

```bash
helm template prod . --debug > templates.yaml
```

- This will dump the template manifests into a file called `templates.yaml` in the current directory
- You can then use this file to debug the templates

## Installation

### Common Steps

#### Create Namespace

- As this cluster could be used for multiple applications with different environments, please create a namespace for the application
- Nomenclature the namespace as `<env>-<app-name>`

```bash
kubectl create namespace prod-botiga-backend
```

#### Set Default Namespace

- This steps is optional
- It simply avoids the need of adding `-n prod-botiga-backend` to every `kubectl` command

```bash
kubectl config set-context --current --namespace=prod-botiga-backend
```

#### Create Secrets

##### Docker Registry Secret

- Secret for pulling images from docker registry of type - `docker-registry`

```bash
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=docker.io \
  --docker-username=varunbotiga \
  --docker-password=your-password \
  --docker-email=varun@botiga.app
```

##### App Secrets

- Upload app confidential information from `.env` file to a secret of type - `generic`
- This approach gives us flexibility to set custom secret values based on environments

```bash
kubectl create secret generic app-secret --from-env-file=.env
```

##### Verifying Secrets

- To verify the secrets, you can use the following command:

```bash
kubectl describe secrets app-secrets
```

- This will not show the actual secret values, but will give you other metadata like when the secret was created.

- To verify the secret values, you can use the following command:

```bash
kubectl get secret docker-registry-secret -o jsonpath="{.data.\.dockerconfigjson}" | base64 --decode
kubectl get secret app-secrets -o jsonpath="{.data.NODE_ENV}" | base64 --decode
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

---

### Install App with a NodePort Service

- *Recommnended only for your local environment*  
- Set following values in your `values.yaml` file:

```yaml
deployment:
  service:
    type: NodePort
    port: 80
    nodePort: 30000
```

- Then, install the chart:

```bash
helm install prod .
```

- If installation is successful and `service.type` is `NodePort`, then, service for Docker-Desktop could be tested at `http://localhost:<node-port>`

---

### Install App with Ingress

- There are 2 major ingresses available:

| Ingress Name               | Description                                              | URL                                                      | Cost                    |
|----------------------------|----------------------------------------------------------|----------------------------------------------------------|-------------------------|
| Kubernetes/ingress-nginx   | Open Source Ingress managed by K8s Community.            | [Link](https://github.com/kubernetes/ingress-nginx)      | Totally Free            |
| Nginx-Ingress              | From Nginx Inc. Free with limited features.              | [Link](https://docs.nginx.com/nginx-ingress-controller)  | Free (Limited) / Nginx+ |

- Going with `Kubernetes/ingress-nginx`, as it's free & does not require any additional setup

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
- Post the changes, follow the [commons steps](#common-steps)

#### Types of Installation

Consider the scenario, where you have 2 following namespaces in same cluster:

- `dev-botiga-app` - For Development Environment
- `prod-botiga-app` - For Production Environment

- Now, you have 2 options to install ingress controller:
  - An Isolated Ingress Controller for each namespace
  - A Shared Ingress Controller for both namespaces

##### Isolated Ingress Controller

- `Not Recommended`
- This could be easily achieved by adding ingress as a dependency in `chart.yaml`

```yaml
dependencies:
  - name: ingress-nginx
    version: "4.8.0" # Specify the version you need
    repository: "https://kubernetes.github.io/ingress-nginx"
    condition: ingress.enabled
```

- Then, run the following command to install the chart:

```bash
helm dependency update .
```

- This will add the ingress chart to your `charts folder` & will also add a `chart.lock` file
- Post this, you have to follow the [common steps](#common-steps) of installation
- `Cons`:
  - More Expensive as multiple ingress controllers are running
  - Harder to manage
  - Also, need some external setup like a load balancer to route traffic to `expected` ingress controller

##### Shared Ingress Controller

- `Recommended`
- In this case, you install a Cluster Wide Ingress Controller
- Create a namespace for ingress controller:

```bash
kubectl create namespace ingress-nginx
```

- Instal ingress controller in this namespace:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install cluster-ingress ingress-nginx/ingress-nginx -n ingress
```

- Now, follow the [common steps](#common-steps) of installation to install dev & prod charts

##### Ingress Controller Overview

The `ingress-nginx` is a versatile ingress controller capable of managing Ingress resources across multiple namespaces.

- **Workings:**
  - `Monitoring`: It's set to watch all namespaces by default, ensuring real-time updates based on Ingress changes across the cluster.
  - `Routing`: Dynamically routes incoming traffic by processing Ingress resource rules, using the underlying NGINX instance.
  - `Isolation`: While it serves multiple namespaces, there's no overlap or interference between rules of different namespaces.
- **Key Benefits:**
  - `Efficiency`: Consolidates resources by eliminating the need for individual controllers in each namespace.
  - `Central Management`: A unified control point makes management, monitoring, and updating straightforward.
  - `Uniformity`: Helps maintain standard configurations, plugins, and templates across different applications.

### Uninstall Chart

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
