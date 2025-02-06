# Helm Exercise 

This Helm chart deploys four main components into a Kubernetes cluster:

1. **Keycloak** – An Identity and Access Management (IAM) server  
2. **PostgreSQL** – A database used by Keycloak to store realms, users, and other data  
3. **PGAdmin** – A database management UI for the same PostgreSQL instance  
4. **Angular Website** – A web application that authenticates against Keycloak

---

## Architecture

1. **Keycloak** connects to **PostgreSQL** using credentials from a shared secret (e.g., `DB_ADDR`, `DB_USER`, `DB_PASSWORD`).  
2. **PGAdmin** can manage that same **PostgreSQL** instance, and it has been pre-configured to access it. The default credentials are also stored in a Kubernetes secret. The default configuration is done through a ConfigMap.  
3. **Angular Website** is a simple app that authenticates against **Keycloak**. The website's configuration is overwritten using a Kubernetes ConfigMap. 

There is optional persistency for the PostgreSQL database, using a Kubernetes Persistent Volume Claim. 

---

## Prerequisites

- **Kubernetes** (v1.23+ recommended)  
- **Helm** (v3 or later)  
- **Docker** (or another container runtime) if you need to build/push images locally  

If you’re deploying locally (e.g., with Minikube or Kind), ensure you can load or pull the necessary images into your cluster.
Also, it's assumed there is an NGINX Ingress Controller configured for your cluster with the "nginx" ingress class. 

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
3. **Install with Helm:**  

   ```bash
   helm upgrade --install my-app .
   ```

By default, the following will happen:
 * Keycloak (with admin credentials from values.yaml) is connected to PostgreSQL (using the same secret-based credentials).
 * PGAdmin is available for managing the same PostgreSQL instance.
 * Website starts up and is available to log in. 
 4. **Import client configuration to Keycloak**
 Before logging in to the application, you need to import the client configuration for the angular website. 
 First, navigate to https://keycloak.local and accept the self-signed certificate. 
 Then, log in using the default credentials: 
 ```
 username: user
 password: bitnami
 ```
<img width="1518" alt="Screenshot 2025-02-06 at 00 19 26" src="https://github.com/user-attachments/assets/aed09a9c-7f11-4070-8841-91ef41f02d48" />
 The admin dashboard will now load. From there, select Realm Settings from the left and then select Partial Import from the action menu on the top right. 
 <img width="1518" alt="Screenshot 2025-02-06 at 00 19 49" src="https://github.com/user-attachments/assets/458276dc-ed5b-4fb9-9418-0f56ea0520fd" />
Then select the file config/realm.json from this repo. 
<img width="1518" alt="Screenshot 2025-02-06 at 00 20 05" src="https://github.com/user-attachments/assets/196142f1-5ae2-4a89-b9f2-3d23c800dc07" />
Then, select the checkbox "7 Clients" and from the dropdown select "Skip" before clicking import. 
If all goes well, you will see the import success window. You can now close this window. 
 <img width="1518" alt="Screenshot 2025-02-06 at 00 20 25" src="https://github.com/user-attachments/assets/d3387580-0dfb-47b4-9864-ff2933afcb13" />

 Now, you can log in to the website using the default user. First, navigate to https://website.local/ which should immediately redirect you to the Keycloak sign-in page.
 <img width="1201" alt="Screenshot 2025-02-06 at 00 22 07" src="https://github.com/user-attachments/assets/e3a3c51c-64f0-4d06-960c-f910ba44b93b" />
Then, you should see this window with the note: isAuthenticated: true. 
 <img width="1201" alt="Screenshot 2025-02-06 at 00 22 13" src="https://github.com/user-attachments/assets/ed8444aa-e00e-41f9-ad57-fb4828092ad2" />

The Helm chart will deploy all the necessary components. Here are some screenshots from the K9s utility, showing the various deployed resources:
<img width="1256" alt="Screenshot 2025-02-06 at 00 16 55" src="https://github.com/user-attachments/assets/d96689c4-d86a-475e-8eca-33b40675333c" />
**Pods**
<img width="1256" alt="Screenshot 2025-02-06 at 00 17 29" src="https://github.com/user-attachments/assets/c1474612-09d2-46ce-8ed3-ed5536d6f78f" />
**Services**
<img width="1256" alt="Screenshot 2025-02-06 at 00 17 19" src="https://github.com/user-attachments/assets/c50c7bc0-e366-4e19-9ccf-79ca827ee041" />
**Persistent Volume Claims**
<img width="1256" alt="Screenshot 2025-02-06 at 00 17 12" src="https://github.com/user-attachments/assets/6be855f7-6388-40bb-a950-19ff29d98603" />
**Deployments**
<img width="1256" alt="Screenshot 2025-02-06 at 00 17 04" src="https://github.com/user-attachments/assets/aa78937d-4747-4e6a-aad0-da9ee512021e" />
**Ingresses**

## Configuration

Global Database Credentials

In values.yaml, you’ll find:

```yaml
globalDatabase:
  username: "keycloak"
  password: "defaultPassword"
  database: "keycloakdb"
```

These are stored in a Kubernetes Secret named `database-credentials`. Keycloak reads them at startup to connect to PostgreSQL.

### Keycloak
* Chart: charts/keycloak/
* Key Settings (e.g., values.yaml):
```yaml
keycloak:
  image: "bitnami/keycloak:latest"
  postgresqlHost: "postgresql.default.svc.cluster.local"
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
* Default credentials:
 ```
 username: admin@example.com
 password: adminpassword
 ```
* The connection to the database should be already configured under Server Group 1 > PostgreSQL1
  <img width="468" alt="Screenshot 2025-02-06 at 01 27 02" src="https://github.com/user-attachments/assets/115062e3-eba8-4779-8334-049b0447e432" />
* The default database password is: `defaultPassword`
* The default keycloak database is: keycloakdb

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
     clientId: 'angular',
     },
   };
```

* The file is mounted into /app/src/environments/environment.ts before the container runs npm start.

You can override any of the configuration by editing the values.yaml file, providing your own values file and passing it to helm

`helm upgrade --install my-app . --values custom.yaml`

or by passing overrides directly to the command line

`helm upgrade --install my-app . --values values.yaml --set keycloak.replicaCount=2`

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

## Potential Improvements
* Make the database deployment into a statefulset.
* Add certmanager to automate the issuance and renewal of certificates. 
* Add metrics server and automatic scaling based on load using HPA. 
