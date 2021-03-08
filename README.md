# kubernetes_en_casa
Un cluster de kubernetes en casa basado en microk8s de Canonical (Ubuntu)

## Prerequisitos
1) Instalar microk8s de Ubuntu
2) Crear alias kubectl de microk8s.kubectl
3) Instalar autocompletado bash Fuente: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## Obtener un certificado Let's Encrypt
 ---- Fuente: https://www.reddit.com/r/kubernetes/comments/g3z5sp/microk8s_with_certmanager_and_letsecncrypt/
Fuente: https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource


Como paso previo debemos permitir el tráfico del puerto 80 y del puerto 443 hacia el host en el cual tenemos el cluster (de host único) de microk8s.

1) Añadimos los addons necesarios n microk8s:
```bash
microk8s enable helm3 ingress
```

2) Instalamos el repositorio de la herramienta de balanceo
```bash
microk8s helm3 repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

3) Instalamos la herramienta ingress-nginx
```bash
microk8s helm3 install quickstart ingress-nginx/ingress-nginx
```
4) Para que el balanceador sea capaz de obtener IPs, como no estamos en un servicio de nube, debemos instalar un proveedor de direcciones IP, en este caso IPs privadas de nuesta LAN. Será necesario conocer un rango de direcciones IP disponibles en la LAN que no se entreguen en el DHCP del router. En mi caso 192.168.1.10-20.
```bash
microk8s enable metallb
```
Veamos el estado del despliegue (Puede tardar un rato): 
```bash
kubectl --namespace default get services -o wide -w quickstart-ingress-nginx-controller
```





Opción de utilización de servicio ingress y opción con certificados:

An example Ingress that makes use of the controller:
```YAML
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls
```

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:
```YAML
  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```














------
2) Instalamos HELM (Ubuntu)

Fuente: https://helm.sh/docs/intro/install/
```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
helm version
```

3) Instalamos el repositorio ingress-nginx (balanceador de tráfico HTTP y HTTPS entrante)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

4) 








---

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

IMPORTANTE: No debemos olvidar modificar el email: CHANGE-ME@example
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


