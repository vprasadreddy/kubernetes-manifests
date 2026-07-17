**NGINX Gateway Fabric — Gateway API Quickstart**

This directory contains examples and instructions to install and test the NGINX Gateway Fabric implementation of the Kubernetes Gateway API.

**Prerequisites**

- **kubectl**: configured to talk to your cluster
- **Helm**: used to install NGINX Gateway Fabric
- **openssl**: to generate test TLS certificates (optional)
- Network access to pull images from the registry and apply CRDs

**1. Verify Helm**
Run:

```
helm version
```

**2. Install Gateway API CRDs**
Apply the upstream Gateway API CRDs used by NGINX Gateway Fabric:

```
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.6.6" | kubectl apply -f -
```

**3. Install NGINX Gateway Fabric**
Install the chart into the `nginx-gateway` namespace:

```
helm install ngf \
	oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
	--namespace nginx-gateway \
	--create-namespace
```

Wait for the primary deployment to become available:

```
kubectl wait --timeout=5m \
	-n nginx-gateway \
	deployment/ngf-nginx-gateway-fabric \
	--for=condition=Available
```

Verify the pods are running:

```
kubectl get pods -n nginx-gateway
```

**4. Create TLS secrets (examples)**
You can create simple self-signed certificates for development/testing with `openssl`.

- Frontend (shop.local):

```
openssl req -x509 -newkey rsa:2048 -keyout frontend-tls.key -out frontend-tls.crt -days 365 -nodes -subj "/CN=shop.local"
kubectl create secret tls shop-cert --cert=frontend-tls.crt --key=frontend-tls.key -n nginx-gateway
```

- Backend (api.local):

```
openssl req -x509 -newkey rsa:2048 -keyout backend-api-tls.key -out backend-api-tls.crt -days 365 -nodes -subj "/CN=api.local"
kubectl create secret tls backend-api-cert --cert=backend-api-tls.crt --key=backend-api-tls.key -n nginx-gateway
```

- Wildcard (development only):

```
openssl req -x509 -newkey rsa:2048 -sha256 -days 365 -nodes -keyout wildcard.key -out wildcard.crt -subj "/CN=_.local" -addext "subjectAltName=DNS:_.local"
kubectl create secret tls wildcard-cert --cert=wildcard.crt --key=wildcard.key -n nginx-gateway
```

List secrets to confirm:

```
kubectl get secret -n nginx-gateway
```

**5. Test the gateway**

- If your gateway exposes a public IP, replace `<gateway-ip>` below.
- Use `curl -k` to ignore TLS verification for self-signed certs during testing.

```
curl -k https://<gateway-ip>/frontend
curl -k https://shop.local
curl -k https://api.local
```

If you are testing with hostnames (e.g., `shop.local`), add entries to your local `/etc/hosts` mapping the gateway IP to the test hostnames (requires sudo):

```
# Example /etc/hosts entries
192.168.139.2  shop.local api.local
```

You can also exercise HTTPRoute resources by sending a `Host` header to the gateway IP:

```
curl -H "Host: shop.local" http://<gateway-ip>
```

**6. Inspect Gateway API resources**

```
kubectl get gatewayclass
kubectl describe gatewayclass <name>
kubectl get gateway -A
kubectl get httproute -A
kubectl get deployments -n nginx-gateway
```

**7. Cleanup**

- Delete gateway resources as needed (example):

```
kubectl delete gateway demo-gateway
```

- Uninstall the chart to remove NGINX Gateway Fabric:

```
helm uninstall ngf -n nginx-gateway
kubectl delete namespace nginx-gateway
```

**Notes & Troubleshooting**

- `curl -k` disables TLS verification — only use for local testing with self-signed certs.
- Replace the example hostnames and IPs with values appropriate for your environment.
- For production use, obtain CA-signed certificates and follow NGINX Gateway Fabric security recommendations.
