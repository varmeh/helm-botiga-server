# SSL at Digital Ocean External Load Balancer

- [SSL at Digital Ocean External Load Balancer](#ssl-at-digital-ocean-external-load-balancer)
  - [1. Set Up DigitalOcean's DNS](#1-set-up-digitaloceans-dns)
  - [2. Deploy `cert-manager`](#2-deploy-cert-manager)
  - [3. Request a Certificate](#3-request-a-certificate)
  - [4. Export the TLS Certificates](#4-export-the-tls-certificates)
  - [5. Add the Certificate to DigitalOcean](#5-add-the-certificate-to-digitalocean)
  - [6. Configure SSL Termination](#6-configure-ssl-termination)
  - [7. Ingress Configuration](#7-ingress-configuration)
  - [8. Monitoring](#8-monitoring)
  - [Conclusion](#conclusion)
  
## 1. Set Up DigitalOcean's DNS

To use Let's Encrypt with the DigitalOcean Load Balancer, you'll first need to make sure your domain's DNS is managed by DigitalOcean.

## 2. Deploy `cert-manager`

Deploy `cert-manager` because even if termination happens at the Load Balancer, you'll still use `cert-manager` to automate the certificate management process.

## 3. Request a Certificate

Instead of using the `ClusterIssuer`, create a Certificate resource directly. This certificate won't be attached to an ingress but instead saved as a Kubernetes Secret.

For `dev.botiga.app`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-botiga-app-certificate
  namespace: your-namespace
spec:
  secretName: dev-botiga-app-tls
  dnsNames:
    - dev.botiga.app
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

Apply the configuration with:

```bash
kubectl apply -f <filename>.yaml
```

## 4. Export the TLS Certificates

Once `cert-manager` has acquired the certificate, export the TLS certificate and key from the secret:

```bash
kubectl get secret dev-botiga-app-tls -n your-namespace -o=jsonpath='{.data.tls\.crt}' | base64 -d > tls.crt
kubectl get secret dev-botiga-app-tls -n your-namespace -o=jsonpath='{.data.tls\.key}' | base64 -d > tls.key
```

## 5. Add the Certificate to DigitalOcean

Go to DigitalOcean dashboard:

- Navigate to **Networking** -> **Load Balancers**.
- Click on your Load Balancer.
- In the **SSL** section, select "Upload a custom SSL certificate" and upload the `tls.crt` and `tls.key` you've just exported.

## 6. Configure SSL Termination

In the same **SSL** section, set the termination to "HTTPS" and select the uploaded certificate.

## 7. Ingress Configuration

Your Ingress resources don't need the TLS section now. The traffic between your Load Balancer and your cluster will be HTTP, as SSL is terminated at the Load Balancer.

## 8. Monitoring

Regularly monitor the certificate using `cert-manager` and update the DO-LB when a new certificate is issued.

## Conclusion

By using this method, you're offloading SSL termination to the DigitalOcean Load Balancer. It simplifies certificate management in the cluster, but you'll need manual steps or automation scripts to update the certificate on the DO Load Balancer whenever `cert-manager` renews it.
