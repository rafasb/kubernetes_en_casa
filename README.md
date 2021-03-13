# kubernetes_en_casa
Un cluster de kubernetes en casa basado en microk8s de Canonical (Ubuntu)

## Prerequisitos
1) Instalar microk8s de Ubuntu (https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#2-deploying-microk8s)
2) (opcional) Crear alias kubectl de microk8s.kubectl
3) (opcional) Instalar autocompletado bash Fuente: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
4) Disponer de un nombre DNS público que acceda a nuestra IP pública del router (Podemos usar https://www.duckdns.org/)
5) Disponer de un medio para la resolución de nombres interna (DNS en LAN). (Podemos usar https://github.com/rafasb/pihole_docker_compose)

## Obtener un certificado Let's Encrypt
 ---- Fuente: https://www.reddit.com/r/kubernetes/comments/g3z5sp/microk8s_with_certmanager_and_letsecncrypt/
Fuente: https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource

### Preparación del cluster

Como paso previo debemos permitir el tráfico del puerto 80 y del puerto 443 hacia el host en el cual tenemos el cluster (de host único) de microk8s. Esto se debe realizar hacia la IP que nos indicará la ejecución de Ingress más adelante.

1) Añadimos los addons necesarios en microk8s:
```bash
microk8s enable helm3 ingress
microk8s enable dns:<IP_DNS_INTERNO>
```

Usaremos <IP_DNS_INTERNO> para permitir la resolución de nombres personalizada con IPs privadas. Se puede usar *microk8s enable dns* si no es necesario tener una resolución interna de DNS.

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

5) Pongamos en marcha un ejemplo de web llamada *kuard*

5.1) Desplegamos los pods
```bash
kubectl apply -f ./ejemplo1/deployment.yml
```
5.2) Desplegamos el servicio
```bash
kubectl apply -f ./ejemplo1/service.yml
```
5.3) Creamos el vínculo ingress para el servicio. OJO: Previamente hay que adecuar el nombre de dominio del host.
```bash
kubectl create -f ./ejemplo1/ingress.yml
```
Para ver el resultado
```bash
kubectl get ingress
```
Es importante tener en consideración el resultado del comando `kubectl get ingress` debemos verificar que el nombre del host empleado en el fichero ./ejemplo1/ingress.yml corresponde con la IP externa concedida o fallará el balanceo. 

La resolución del nombre en Internet debe permitir alcanzar la IP pública del router para obtener el certificado. Si internamente queremos testear el resultado, debemos hacer que la IP enterna del LoadBalancer sea resuelta con el mismo nombre empleado en el fichero. Por tanto tendremos una IP privada en el resultado del LoadBalancer y su correspondiente resolución dentro de nuestra LAN y una resolución externa (p.e. DuckDNS) con la IP pública de nuestro router. 

Obviamente en el router debemos hacer que cualquier petición hacia el puerto 80 y 443 se encamine a la IP externa (privada) mostrada por el comando `kubectl get ingress`

5.4) Verificación y borrado

*Podemos verificar el funcionamiento accediendo a la url http://\<tu_propio_FQDN\>*

kubectl delete -f ejemplo1/ingress.yml
kubectl delete -f ejemplo1/service.yml
kubectl delete -f ejemplo1/deployment.yml

### Instalación de cert-manager

Fuente: https://cert-manager.io/docs/installation/kubernetes/#installing-with-regular-manifests

1) Instalar el CRD, cert-manager, namespace y webhook con una sola declaración YAML:
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
```
o
```bash
kubectl apply -f certmanager/cert-manager.yml
```

2) Verificamos con la creación de un Issuer para probar. Una vez más, hay que personalizar el valor de dnsNames
```bash
kubectl apply -f certmanager/test-resources.yml
kubectl describe certificate -n cert-manager-test
```

3) Eliminamos el Issuer de pruebas
```bash
kubectl delete -f certmanager/test-resources.yml
```

### Despliegue de certificados con Issuer

Se empleará Issuer (junto con el namespace), y no ClusterIssuer, para certificados empleados en usos no generales al cluster.
Emplearemos ClusterIssuer para certificados que pueden ser empleados por los namespaces del cluster.
Modifica el fichero issuer/staging-issuer.yml con el correo electrónico apropiado.

TODO : Crear cuenta asociada al correo.

```bash
kubectl apply -f issuer/staging-issuer.yml [--namespace default]
```

Se puede verificar el estado mediante:
```bash
kubectl describe issuer letsencrypt-staging [--namespace default]
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


