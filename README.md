# Umbrella Helm Chart

This Helm chart deploys four main components into a Kubernetes cluster:

1. **Keycloak** – An Identity and Access Management (IAM) server  
2. **PostgreSQL** – A database used by Keycloak to store realms, users, and other data  
3. **PGAdmin** – A database management UI for the same PostgreSQL instance  
4. **Angular Website** – A web application that authenticates against Keycloak

---

## Architecture

1. **Keycloak** connects to **PostgreSQL** using credentials from a shared secret (e.g., `DB_ADDR`, `DB_USER`, `DB_PASSWORD`).  
2. **PGAdmin** can manage that same **PostgreSQL** instance, and it has been pre-configured to access it.  
3. **Angular Website** is a simple app that authenticates against **Keycloak**.

---

## Prerequisites

- **Kubernetes** (v1.23+ recommended)  
- **Helm** (v3 or later)  
- **Docker** (or another container runtime) if you need to build/push images locally  

If you’re deploying locally (e.g., with Minikube or Kind), ensure you can load or pull the necessary images into your cluster.

---

## Installation

1. **Build/Push the web app image** (if needed):
Clone the web app git repository and build the Docker image:
   ```bash
   docker build -t tributech-ui-oauth-sample:latest .
   # Optional: minikube image load tributech-ui-oauth-sample:latest
   # or docker push tributech-ui-oauth-sample:latest
   ```
   
2. **Clone or copy this Helm chart folder to your local machine.**
3.	**Install with Helm:**  

   ```bash
   helm upgrade --install my-app ./my-umbrella-chart
   ```

By default, the following happen:
	* Keycloak (with admin credentials from values.yaml) is connected to PostgreSQL (using the same secret-based credentials).
	* PGAdmin is available for managing the same PostgreSQL instance (manual server entry).
	* Website is an Angular app that compiles on container startup, using a default environment.ts.

## Configuration

Global Database Credentials

In values.yaml, you’ll find:

```yaml
globalDatabase:
  username: "keycloak"
  password: "keycloakpassword"
  database: "keycloakdb"
```

These are stored in a Kubernetes Secret named <release>-db-secret. Keycloak reads them at startup to connect to PostgreSQL.

### Keycloak
* Chart: charts/keycloak/
* Key Settings (e.g., values.yaml):
```yaml
keycloak:
  image: "bitnami/keycloak:latest"
  postgresqlHost: "my-umbrella-chart-postgresql"
  auth:
   adminUser: "admin"
   adminPassword: "adminpassword"
```
* Environment variables for DB credentials come from the secret db-secret.

### PostgreSQL
* Chart: charts/postgresql/
* Optional Persistence: Enabled by default; controlled by setting:
```yaml
postgresql:
 persistence:
  enabled: true
  size: "2Gi"
  storageClass: "standard"
```
* Service Name is typically <release>-postgresql. Keycloak’s postgresqlHost references this service.

### PGAdmin
* Chart: charts/pgadmin/
* Manual connection to the same Postgres server:
	1. Go to http://pgadmin.local
	2. Login with the credentials in pgadmin  (e.g., admin@example.com / adminpassword)
	3. Add a new server, specifying:
* Host: my-umbrella-chart-postgresql (or <release>-postgresql)
* Port: 5432
* Username: keycloak
* Password: keycloakpassword
* Database: keycloakdb (optional)

### Angular Website
* Chart: charts/website/
* environment.ts is stored as a multiline string in values.yaml:
```yaml
website:
  environmentTs: |
   export const environment = {
   production: false,
    auth: {
     authority: 'http://keycloak.local/realms/node',
     clientId: 'dataspace-admin',
     },
   };
```
* Override it at install time:
```yaml
helm upgrade --install my-app ./my-umbrella-chart \
  --set-file website.environmentTs=./my-environment.ts
```
* The file is mounted into /app/src/environments/environment.ts before the container runs npm start.

## Accessing Services
1. Ingress Hostnames: By default:
	* Keycloak: keycloak.local
	* PGAdmin: pgadmin.local
	* Website: website.local
2. Add them to /etc/hosts:
```
<MINIKUBE_IP>  keycloak.local
<MINIKUBE_IP>  pgadmin.local
<MINIKUBE_IP>  website.local
```
(Replace <MINIKUBE_IP> with the IP from minikube ip or similar.)

3. Visit:
	* Keycloak: https://keycloak.local
	* PGAdmin: http://pgadmin.local
	* Website: https://website.local

## Testing

This chart includes a helm test Pod that curls Keycloak and the Website:
```sh
helm test my-app
```
If both return 200 OK, you’ll see “All tests passed!”.

## Uninstalling
```sh
helm uninstall my-app
```
This removes all Pods, Services, Ingress, etc. If persistence was enabled, you may also have to manually remove any leftover PVCs (PersistentVolumeClaims).

##Summary
* Keycloak + PostgreSQL for identity/data storage
* PGAdmin for manual DB management of the same Postgres instance
* Angular Website with a runtime-override of environment.ts (no need to rebuild the Docker image for different Keycloak realms/URLs)
