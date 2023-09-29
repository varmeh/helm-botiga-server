# SSL Using Ingress Controller

- [SSL Using Ingress Controller](#ssl-using-ingress-controller)
  - [1. Install Cert-Manager](#1-install-cert-manager)
  - [2. Create a Let's Encrypt Issuer or ClusterIssuer](#2-create-a-lets-encrypt-issuer-or-clusterissuer)
  - [3. Configure Ingress to Use Automated Certificates](#3-configure-ingress-to-use-automated-certificates)
  - [4. DNS Configuration](#4-dns-configuration)
  - [5. Monitor Certificate Issuance](#5-monitor-certificate-issuance)
  - [Conclusion](#conclusion)

Sure, you can automate SSL certificate provisioning for your `dev.botiga.app` domain using `Let's Encrypt` and `Cert-Manager` in your Kubernetes cluster. Here's a step-by-step guide:

## 1. Install Cert-Manager

Cert-Manager will automatically handle certificate requests and renewals for you.

First, add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
```

Update your helm repositories:

```bash
helm repo update
```

Install the Cert-Manager Helm chart:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.7.1 \
  --set installCRDs=true
```

## 2. Create a Let's Encrypt Issuer or ClusterIssuer

You can choose between an `Issuer` (namespace-scoped) and a `ClusterIssuer` (cluster-scoped). For simplicity, we'll use `ClusterIssuer`:

Save the following YAML into a file named `cluster-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Your email address (replace with your email)
    email: your-email@example.com 
    # Name of a secret used to store the account's private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

Replace `your-email@example.com` with your email.

Apply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

## 3. Configure Ingress to Use Automated Certificates

Update your Ingress configuration to request certificates from the `ClusterIssuer`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-botiga-backend
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
  - host: dev.botiga.app
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dev-botiga-backend
            port:
              number: 80
  tls: # <- Adding TLS block
  - hosts:
    - dev.botiga.app
    secretName: dev-botiga-app-tls
```

This Ingress resource now requests a certificate from Let's Encrypt via the specified `ClusterIssuer`. The certificate will be stored in a secret named `dev-botiga-app-tls`.

Once you apply this ingress with `kubectl apply -f <your-ingress-file.yaml>`, Cert-Manager will request a certificate from Let's Encrypt. It'll use the HTTP-01 challenge mechanism, creating a temporary ingress rule to prove you control the domain.

## 4. DNS Configuration

Make sure that your domain `dev.botiga.app` points to the external IP of your Ingress controller (or DigitalOcean LoadBalancer in front of it). Let's Encrypt needs to reach your cluster to verify the domain ownership.

## 5. Monitor Certificate Issuance

You can check the status of your certificate request:

```bash
kubectl describe certificate dev-botiga-app-tls
```

Once the certificate is issued, it will automatically be renewed by Cert-Manager as it approaches expiration.

## Conclusion

By following these steps, you'll have an automated Let's Encrypt certificate for `dev.botiga.app` in your DigitalOcean Kubernetes cluster. It'll be secured and auto-renewed, ensuring uninterrupted HTTPS access.
