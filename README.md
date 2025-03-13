# ChirpStack Deployment Lifecycle

## 0 cd To Directory

```sh
cd files
```
## 1. Create Namespace & ConfigMap & StorageClass
Create the necessary Kubernetes namespaces for ChirpStack components and apply the configurations.
```sh
kubectl create ns chirpstack
kubectl apply -f config-maps
kubectl apply -f storage.yml
```

## 2. Install Redis
Deploy the ChirpStack Redis component using Helm charts.
```sh
helm upgrade --install chirpstack-redis oci://registry-1.docker.io/bitnamicharts/redis \
    -f ./values/redis-values.yaml --namespace chirpstack-redis --create-namespace
```
- If you want to update your redis yaml, take it from here https://github.com/bitnami/charts/blob/main/bitnami/redis/values.yaml. In this yaml there is no confuguration but password at line 147 auth.password section. 
## 3. Install PostgreSQL
Deploy the ChirpStack PostgreSQL component using Helm charts.
Additionally, the following configuration is used for PostgreSQL:
```yaml
scripts:
  create_databases.sql: |
    CREATE DATABASE chirpstack;
    \c chirpstack;
    create extension pg_trgm;
    create extension hstore;
```
```sh
helm upgrade --install chirpstack-postgres oci://registry-1.docker.io/bitnamicharts/postgresql \
    -f ./values/postgres-values.yaml --namespace chirpstack-postgres --create-namespace
```
- If you want to update your postgres yaml, take it from here https://github.com/bitnami/charts/blob/main/bitnami/postgresql/values.yaml but do **NOT** forget to add the chirpstack database scripts in yaml at line 353 primary.scripts section.

### Note on PostgreSQL Operator and CloudNativePG
ChirpStack does **not** work with PostgreSQL Operator or CloudNativePG due to permission restrictions. These solutions do **not** grant the necessary privileges for creating databases, extensions, and roles required by ChirpStack.

## 4. Apply Services and Deployments
Apply the necessary Kubernetes service and deployment files for ChirpStack.
```sh
kubectl apply -f services
kubectl apply -f deployments
```

## 5. Deploy ChirpStack

```sh
cd ..
cd chirpstack
```

Before installation do **NOT** forget to create a api key for chripstack core with this command.

```sh
openssl rand -base64 32
```
```sh
apiSecret
```
Also do **NOT** forget to add your postgres and redis **credits&hosts** in yaml:
for postgres at line 45
for redis at line 67


Change the value of apiSecret at chirpstack/values.yaml line 31.

For applying the regions for Chirpstack use this command.
```sh
kubectl apply -k ./
```
You can easly delete or add new regions in the "kustomization.yaml" file by just deleting the toml file name.

Install the ChirpStack Helm chart.

```sh
helm upgrade --install chirpstack ./ -n chirpstack --create-namespace
```
## 6. Deploy Ingress (If Needed)

```sh
cd .. 
cd files
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    -f ./values/ingress.yaml --namespace chirpstack
```
In values at line 1216 tcp section there is a configuration for mqtt and chirpstack. Also dont forget to change your dns in deployment/ingress.yaml
```sh
tcp: 
  "8883": "chirpstack/mosquitto:8883:PROXY"
  "8080": "chirpstack/chirpstack-core:8080:PROXY"
```
```sh
spec:
  ingressClassName: nginx
  rules:
    - host: DNS_NAME
```
## 7. Section of Uninstalling ChirpStack
To remove ChirpStack and its dependencies, use the following commands:

```sh
# Uninstall the ChirpStack application
helm uninstall chirpstack -n chirpstack

# Remove services and deployments
kubectl delete -f services -n chirpstack
kubectl delete -f deployments -n chirpstack
kubectl delete -f config-maps -n chirpstack
# Uninstall Redis and PostgreSQL components
helm uninstall chirpstack-redis -n chirpstack-redis
helm uninstall chirpstack-postgres -n chirpstack-postgres

# Delete namespaces
kubectl delete namespace chirpstack
kubectl delete namespace chirpstack-redis
kubectl delete namespace chirpstack-postgres
```

---
This part of document provides a step-by-step guide on how to deploy and uninstall the ChirpStack platform in a Kubernetes environment. To verify that all components are successfully running, use the following commands:

```sh
kubectl get pods -n chirpstack
kubectl get svc -n chirpstack
```

For troubleshooting, you can use the `kubectl logs` and `kubectl describe` commands.



# Architecture

## 1. Architecture Overview

In this setup, the **ChirpStack-core** pod serves as both the **Application Server** and the **Network Server**. The message traffic between components primarily flows through **Mosquitto (MQTT)**, **PostgreSQL**, and **Redis**.

The system consists of the following main components:

- **LoRa Gateway** (Physical device, **not** a Kubernetes pod)
- **Mosquitto** (MQTT Broker)
- **ChirpStack Gateway Bridge**
- **ChirpStack Core (Network Server + Application Server)**
- **PostgreSQL** (Database)
- **Redis** (Cache and session storage)
- **Geolocation Server** (If using TDOA/RSSI-based geolocation)
---

## 2. Data Flow 

Here’s how data travels from the LoRa device to the end-user:

###  1. LoRaWAN Gateway → MQTT Broker (Mosquitto)
- The LoRa device sends a data packet.
- The gateway receives the packet and forwards it to the MQTT Broker (Mosquitto).
- **MQTT Topic Example:**
---

###  2. MQTT Broker → ChirpStack Gateway Bridge
- Mosquitto forwards the message to **chirpstack-gateway-bridge**.
- Gateway Bridge converts LoRa packets into **JSON or Protobuf**.
- It then forwards data via MQTT to **chirpstack-core**.
---
### 3. ChirpStack Core (Network Server)
- **Network Server** verifies **LoRaWAN MIC (Message Integrity Code)**.
- Checks device authentication (**DevEUI, AppEUI, AppKey**).
- If valid, processes **OTAA/ABP activation** and MAC commands.
- Determines which gateway received the signal.
- If verification fails, **Network Server rejects the message**.

---

###  4. PostgreSQL and Redis for Data Storage

ChirpStack-core stores data in:

1. **PostgreSQL:**
 - Device info, LoRaWAN sessions, user details.
 - ChirpStack UI settings.
 - **Pod:** `chirpstack-postgres-postgresql`

2. **Redis:**
 - Temporary LoRaWAN session data, MAC commands.
 - Network Server uses it for fast read/write.
 - **Pod:** `chirpstack-redis-master`

---

###  5. ChirpStack Core (Application Server + Geolocation Server + Network Server)
- **Application Server** processes verified data.
- Converts data to JSON or other formats.
- Forwards data via MQTT, HTTP, Kafka, or gRPC.
- If downlink is required, it sends data to **Network Server**.
- If devices lack GPS, **chirpstack-geolocation-server** estimates location using RSSI/TDOA.
- **Pod:** `chirpstack-geolocation-server`


---

## Component Connections (Summary Table)

| **Source**                     | **Destination**                 | **Protocol**  |
|--------------------------------|--------------------------------|--------------|
| **LoRaWAN Gateway**            | Mosquitto (MQTT Broker)        | UDP / MQTT   |
| **MQTT Broker (Mosquitto)**    | ChirpStack Gateway Bridge      | MQTT         |
| **ChirpStack Gateway Bridge**  | ChirpStack-core (Network Server) | MQTT      |
| **ChirpStack-core (Network Server)** | Redis | TCP |
| **ChirpStack-core (Network Server)** | PostgreSQL | TCP |
| **ChirpStack-core (Network Server)** | ChirpStack-core (Application Server) | Internal (Pod) |
| **ChirpStack-core (Application Server)** | PostgreSQL | TCP |
| **ChirpStack-core (Application Server)** | MQTT / HTTP / gRPC / Kafka | MQTT / REST API |
| **ChirpStack Geolocation Server** | ChirpStack-core (Network Server) | gRPC / HTTP |

---

## Conclusion 

- **LoRaWAN data follows the path:**  
`Mosquitto → ChirpStack Gateway Bridge → Network Server → Application Server`
- **ChirpStack-core acts as both the Network and Application Server.**
- **PostgreSQL stores persistent data, Redis stores temporary session data.**
- **Geolocation Server estimates device position.**
- **Downlink messages follow the reverse path.**

---
