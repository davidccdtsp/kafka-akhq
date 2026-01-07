# Despliegue local Kafka, akhq y keycloack

## Kafka
Despliegue de kafka sobre minikube

```bash
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Cracion del cluster
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka 
```

## Keycloak

```bash
# Habilitar el ingress en el cluster si no lo estuviese
minikube addons enable ingress

# Creacion keycloak
kubectl create -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/refs/heads/main/kubernetes/keycloak.yaml -n kafka

# Creacion del ingress controller para keycloak
# Crear el secreto TLS en el namespace kafka usando tus archivos
kubectl create secret tls keycloak-tls-secret \
  --cert=keycloak.crt \
  --key=keycloack.key \
  -n kafka

kubectl apply -n kafka -f ./ingress/keycloak.yaml
```

```bash
KEYCLOAK_URL=https://keycloak.$(minikube ip).nip.io &&
echo "" &&
echo "Keycloak:                 $KEYCLOAK_URL" &&
echo "Keycloak Admin Console:   $KEYCLOAK_URL/admin" &&
echo "Keycloak Account Console: $KEYCLOAK_URL/realms/myrealm/account" &&
echo ""
```

Llegados a este punto necesario configurar parcialmente el cliente en keycloak para obtener el client scret y substituirlo en la linea 81 del values.yaml.

## Akhq



### Prerequisitos
Para habilitar la conexion local mediante ssl debemos crear dos certificados, el empleado por el frontend de akhq y el trustore con el certificado de keycloak para Java.

```bash

# Generamos el secreto
openssl req -x509 -newkey rsa:4096 -sha256 -nodes   -keyout akhq.key -out akhq.crt   -days 365   -subj "/CN=akhq.$(minikube ip).nip.io"   -addext "subjectAltName=DNS:akhq.$(minikube ip).nip.io"


kubectl create secret tls akhq-tls-secret --cert=akhq.crt --key=akhq.key -n kafka

# Trustore para java
openssl req -x509 -newkey rsa:4096 -sha256 -nodes   -keyout keycloak.key -out keycloak.crt   -days 365   -subj "/CN=keycloak.$(minikube ip).nip.io"   -addext "subjectAltName=DNS:keycloak.$(minikube ip).nip.io"

keytool -import -alias keycloak \
  -file keycloak.crt \
  -keystore truststore.jks \
  -storepass changeit \
  -noprompt

kubectl create secret generic akhq-truststore --from-file=truststore.jks=./truststore.jks -n kafka

```


```bash
helm upgrade --install akhq-local . -n kafka -f ./values.yaml
```


