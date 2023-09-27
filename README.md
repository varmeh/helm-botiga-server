# Helm Chart for Botiga Backend

- [Helm Chart for Botiga Backend](#helm-chart-for-botiga-backend)
  - [Debug Templates](#debug-templates)
  - [Installation](#installation)
    - [Create the namespace](#create-the-namespace)
    - [Set Default Namespace](#set-default-namespace)
    - [Create Secrets](#create-secrets)
      - [Docker Registry Secret](#docker-registry-secret)
      - [App Secrets](#app-secrets)
      - [Verifying Secrets](#verifying-secrets)
    - [Mounting Firebase SDK File](#mounting-firebase-sdk-file)
    - [Install Chart](#install-chart)
    - [Upgrading Chart](#upgrading-chart)
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

### Create the namespace

- As this cluster could be used for multiple applications with different environments, please create a namespace for the application
- Nomenclature the namespace as `<env>-<app-name>`

```bash
kubectl create namespace prod-botiga-backend
```

### Set Default Namespace

- This steps is optional
- It simply avoids the need of adding `-n prod-botiga-backend` to every `kubectl` command

```bash
kubectl config set-context --current --namespace=prod-botiga-backend
```

### Create Secrets

#### Docker Registry Secret

- Secret for pulling images from docker registry of type - `docker-registry`

```bash
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=docker.io \
  --docker-username=varunbotiga \
  --docker-password=your-password \
  --docker-email=varun@botiga.app
```

#### App Secrets

- Upload app confidential information from `.env` file to a secret of type - `generic`
- This approach gives us flexibility to set custom secret values based on environments

```bash
kubectl create secret generic app-secret --from-env-file=.env
```

#### Verifying Secrets

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

### Mounting Firebase SDK File

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

### Install Chart

```bash
helm install prod .
```

- If installation is successful and `service.type` is `NodePort`, then, service for Docker-Desktop could be tested at `http://localhost:<node-port>`

### Upgrading Chart

```bash
helm upgrade prod .
```

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
