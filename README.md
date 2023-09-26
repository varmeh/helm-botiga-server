# Helm Chart for Botiga Backend

- [Helm Chart for Botiga Backend](#helm-chart-for-botiga-backend)
  - [Debug Templates](#debug-templates)
  - [Installation Steps](#installation-steps)
    - [Create the namespace](#create-the-namespace)
    - [Default Namespace](#default-namespace)
    - [Create Secrets](#create-secrets)
      - [Docker Registry Secret](#docker-registry-secret)
      - [App Secrets](#app-secrets)
      - [Verifying Secrets](#verifying-secrets)
      - [Mounting Firebase SDK File](#mounting-firebase-sdk-file)

## Debug Templates

- To debug the templates, you can use the following command:

```bash
helm template prod . --debug > templates.yaml
```

- This will dump the template manifests into a file called `templates.yaml` in the current directory
- You can then use this file to debug the templates

## Installation Steps

### Create the namespace

- As this cluster could be used for multiple applications with different environments, please create a namespace for the application
- Nomenclature the namespace as `<env>-<app-name>`

```bash
kubectl create namespace prod-botiga-backend
```

### Default Namespace

- Set above created namespace as default namespace for the current context

```bash
kubectl config set-context --current --namespace=prod-botiga-backend
```

- **OR**

- Add `-n prod-botiga-backend` to access the resources in the namespace

### Create Secrets

#### Docker Registry Secret

- Secret for pulling images from docker registry

```bash
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=docker.io \
  --docker-username=varunbotiga \
  --docker-password=your-password \
  --docker-email=varun@botiga.app
```

#### App Secrets

- Upload app confidential information from `.env` file to a secret
- This approach gives us flexibility to set custom secret values based on environments

```bash
kubectl create secret generic app-secrets --from-env-file=.env
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

#### Mounting Firebase SDK File

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
