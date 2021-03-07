# kubernetes_en_casa
Un cluster de kubernetes en casa basado en microk8s de Canonical (Ubuntu)

## Obtener un certificado Let's Encrypt
Fuente: https://www.reddit.com/r/kubernetes/comments/g3z5sp/microk8s_with_certmanager_and_letsecncrypt/
Fuente: https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource

Como paso previo debemos permitir el tráfico del puerto 80 y del puerto 443 hacia el host en el cual tenemos el cluster (de host único) de microk8s.

1) Añadimos los addons necesarios n microk8s:
```bash
microk8s enable helm3 ingress
```
2) Utilizamos helm para cargar el repositorio con el gestor de certificados y para instalar el gestor:
```bash
microk8s kubectl create namespace cert-manager
microk8s helm3 repo add jetstack https://charts.jetstack.io
microk8s helm3 repo update
microk8s helm3 install cert-manager jetstack/cert-manager \
  --namespace cert-manager --version v0.15.2 \
  --set installCRDs=true \
  --set ingressShim.defaultIssuerName=letsencrypt-production \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  --set ingressShim.defaultIssuerGroup=cert-manager.io
```
4) Modificamos la configuración del cluster para incluir el gestor de certificados:
```bash
microk8s kubectl apply -f - <<YAML
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: CHANGE-ME@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production-issuer-account-key
    solvers:
    - selector: {}
      http01:
        ingress:
          class: nginx
YAML
```

6) 


