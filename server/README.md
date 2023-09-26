<img src="https://avatars3.githubusercontent.com/u/12783832?s=200&v=4" height="100" width="100" /><img src="https://avatars3.githubusercontent.com/u/15859888?s=200&v=4" width="100" height="100"/>

# Khulnasoft Security Server Helm Chart

These are Helm charts for installation and maintenance of Khulnasoft Container Security Platform Console Server, Gateway, Database.

## Contents

- [Khulnasoft Security Server Helm Chart](#khulnasoft-security-server-helm-chart)
  - [Contents](#contents)
  - [Prerequisites](#prerequisites)
    - [Container Registry Credentials](#container-registry-credentials)
    - [Service Account](#service-account)
    - [PostgreSQL database](#postgresql-database)
  - [Installing the Chart](#installing-the-chart)
    - [Installing Khulnasoft Web from Helm Private Repository](#installing-khulnasoft-web-from-helm-private-repository)
  - [Advanced Configuration](#advanced-configuration)
  - [Envoy](#envoy)
  - [Database](#database)
  - [Configuring HTTPS for Khulnasoft's server](#configuring-https-for-khulnasofts-server)
  - [Configuring mTLS/TLS for Khulnasoft Server and Khulnasoft Gateway](#configuring-mtlstls-for-khulnasoft-server-and-khulnasoft-gateway)
    - [Create Root CA (Done once)](#create-root-ca-done-once)
    - [Create the certificate key for Khulnasoft server, gateway](#create-the-certificate-key-for-khulnasoft-server-gateway)
    - [Create secrets with generated certs and change `values.yaml` as mentioned below](#create-secrets-with-generated-certs-and-change-valuesyaml-as-mentioned-below)
  - [Configure mTLS between server/gateway and external DB](#configure-mtls-between-servergateway-and-external-db)
  - [Setting active-active mode](#setting-active-active-mode)
  - [Configurable Variables](#configurable-variables)
  - [Issues and feedback](#issues-and-feedback)

## Prerequisites

### Container Registry Credentials

[Link](../docs/imagepullsecret.md)

### Service Account
By default the chart will create a SA named `khulnasoft-sa`, and will deploy the resources using it.
There are 3 scenarios for the service account usage:

1. Using the default SA `khulnasoft-sa`, this is the default option, no additional values need to be provided.


2. Using a custom name for SA, in this scenario the following values need to be provided:
   1. `serviceAccount.name` and `gateway.serviceAccount.name` containing the desired name.


3. Using an existing SA or imperatively created one:
   1. For example the SA can be created and patched with this commands:

    ```shell
   # Create the ServiceAccount
    kubectl create serviceaccount khulnasoft-custom-sa -n khulnasoft
   # Patch the ServiceAccount with the pull secret
    kubectl patch serviceaccount khulnasoft-custom-sa -n khulnasoft -p '{"imagePullSecrets": [{"name": "khulnasoft-registry-secret"}]}'
    ```
    and the values that need to be provided are:
   1. `serviceAccount.name` and `gateway.serviceAccount.name` containing the value `khulnasoft-custom-sa`.
   2. `serviceAccount.create` with the value`false`

### PostgreSQL database

Khulnasoft Security recommends implementing a highly-available PostgreSQL database. By default, the console chart will install a PostgreSQL database and attach it to persistent storage for POC usage and testing. For production use, one may override this default behavior and specify an existing PostgreSQL database by setting the following variables in values.yaml:

```yaml
global:
  db:
    external:
      enabled: true
      name: example-khulnasoft
      host: khulnasoft-db
      port: 5432
      user: khulnasoft-db-username
      password: verysecret
```
## Installing the Chart
Follow the steps in this section for production grade deployments.
You can either clone khulnasoft-helm git repo or you can add our helm private repository ([https://helm.khulnasoft.com](https://helm.khulnasoft.com))

### Installing Khulnasoft Web from Helm Private Repository

* Add Khulnasoft Helm Repository
```shell
helm repo add khulnasoft-helm https://helm.khulnasoft.com
helm repo update
```

* Check for available chart versions either from [Changelog](./CHANGELOG.md) or by running the below command
```shell
helm search repo khulnasoft-helm/server --versions
```

* Install Khulnasoft

```shell
helm upgrade --install --namespace khulnasoft <RELEASE_NAME> khulnasoft-helm/server --set imageCredentials.username=<>,imageCredentials.password=<>,global.platform=<>
```

## Advanced Configuration

## Envoy

   In order to support L7 / gRPC communication between gateway and enforcers Khulnasoft recommends that customers use the Envoy load balancer. Following are the detailed steps to enable and deploy a secure envoy based load balancer.

   1. Generate TLS certificates signed by a public CA or Self-Signed CA

      ```shell
      # Self-Signed Root CA (Optional)
      #####################################################################################

      # Create Root Key
      # If you want a non password protected key just remove the -des3 option
      openssl genrsa -des3 -out rootCA.key 4096

      # Create and self sign the Root Certificate
      openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt

      #####################################################################################
      # Create a certificate
      #####################################################################################

      # Create the certificate key
      openssl genrsa -out mydomain.com.key 2048
      # Create the signing (csr)
      openssl req -new -key mydomain.com.key -out mydomain.com.csr
      # Verify the csr content
      openssl req -in mydomain.com.csr -noout -text

      #####################################################################################
      # Generate the certificate using the mydomain csr and key along with the CA Root key
      #####################################################################################

      openssl x509 -req -in mydomain.com.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out mydomain.com.crt -days 500 -sha256

      #####################################################################################
      # If you wish to use a Public CA like GoDaddy or LetsEncrypt please
      # submit the mydomain csr to the respective CA to generate mydomain crt
      ```

   2. Create TLS cert secret

      ```shell
      kubectl create secret generic khulnasoft-lb-tls --from-file=mydomain.com.crt --from-file=mydomain.com.key --from-file=rootCA.crt -n khulnasoft
      ```

   3. Edit the values.yaml file to include above secret
   ```
       TLS:
         listener:
            secretName: "khulnasoft-lb-tls"
            publicKey_fileName: "mydomain.com.crt"
            privateKey_fileName: "mydomain.com.key"
            rootCA_fileName: "rootCA.crt"
   ```

   4. [Optional] If Gateway requires client certificate authentication edit the values.yaml to include those secrets as well:
   ```
       TLS:
         ...
         cluster:
            enabled: true
            secretName: "khulnasoft-lb-tls-custer"
            publicKey_fileName: "envoy.crt"
            privateKey_fileName: "envoy.key"
            rootCA_fileName: "rootCA.crt"
   ```

   5. For more customizations please refer to [***Configurable Variables***](#configure-variables)

## Database

   1. By default khulnasoft helm chart will deploy a database container. If you wish to use an external database please set `db.external.enabled` to true and the following with appropriate values.
      ```shell
      global.db.external.name
      global.db.external.host
      global.db.external.port
      global.db.external.user
      global.db.external.password
      ```
   2. By default same database (Packaged DB Container | Managed DB like AWS RDS) will be used to host both main DB and Audit DB. If you want to use a different database for audit db then set following variables in the values.yaml file
      ```shell
      global.db.external.auditName
      global.db.external.auditHost
      global.db.external.auditPort
      global.db.external.auditUser
      global.db.external.auditPassword
      ```
   3. If you are using packaged DB container then
      1. AQUA_ENV_SIZE variable can be used to define the sizing of your DB container in terms of number of connections and optimized configuration but not the PV size. Please choose appropriate PV size as per your requirements.
      2. By default AQUA_ENV_SIZE is set to `"S"` and the possible values are `"M", "L"`

## Configuring HTTPS for Khulnasoft's server

   By default Khulnasoft will generate a self signed cert and will use the same for HTTPS communication. If you wish to use your own SSL/TLS certs you can do this in two different ways

   1. Ingress(optional): Kubernetes ingress controller can be used to publicly expose khulnasoft web UI over a HTTPS connection

   2. LoadBalancer(Default):
      1. Create Kubernetes secrets for server component using the respective SSL certificates.
         ```shell
         kubectl create secret generic khulnasoft-web-certs --from-file khulnasoft_web.key --from-file khulnasoft_web.crt --from-file rootCA.crt -n khulnasoft

         here: khulnasoft_web.key    = privateKey
               khulnasoft_web.crt    = publicKey
               rootCA.crt      = rootCA
               khulnasoft-web-certs  = secret Name
         ```

      2. Enable `web.TLS.enable `to `true` in values.yaml
      3. Add the certificates secret name `web.TLS.secretName` in values.yaml
      4. Add respective certificate file names to `web.TLS.publicKey_fileName`, `web.TLS.privateKey_fileName` and `web.TLS.rootCA_fileName`(Add rootCA if certs are self-signed) in values.yaml
      5. Proceed with the deployment.

## Configuring mTLS/TLS for Khulnasoft Server and Khulnasoft Gateway
   By default, deploying Khulnasoft Enterprise configures TLS-based encrypted communication, using self-signed certificates, between Khulnasoft components. If you want to use self-signed certificates to establish mTLS between khulnasoft components use the below instrictions to generate rootCA and component certificates

   > **Note:** **_mTLS communication and setup is only supported for self-hosted Khulnasoft. It is not supported for Khulnasoft ESE and Khulnasoft SAAS_**

   ### Create Root CA (Done once)

   ***Important:*** Optional, If you have already the existing rootCA certs you can skip to generating [component certificates] step.(#create-the-certificate-key-for-khulnasoft-server-gateway)

   **1. Create Root Key**

   **Attention:** this is the key used to sign the certificate requests, anyone holding this can sign certificates on your behalf. So keep it in a safe place!
   - Generating rootCA key
      ```shell
      openssl genrsa -des3 -out rootCA.key 4096
      ```

   If you want a non password protected key just remove the `-des3` option

   **2. Create and self sign the Root Certificate**
   - Creating rootCA certificate
      ```shell
      openssl req -new -x509 -days 365 -key rootCA.key -sha256 -days 1024 -out rootCA.crt
      ```
   ### Create the certificate key for Khulnasoft server, gateway
   After creating rootCA certs start generating component certificates.


   **3. Create keys and signing (csr):**

   The certificate signing request is where you specify the details for the certificate you want to generate.
   This request will be processed by the owner of the Root key (you in this case since you create it earlier) to generate the certificate.

   ***Important:*** Please mind that while creating the signign request is important to specify the `Common Name` providing the IP address or domain name for the service, otherwise the certificate cannot be verified.

   ***Important for multi-cluster:*** When using multi-cluster setup like enforcer/kube-enforcers from various clusters or connecting over the internet with gateway/envoy, then `SAN(subjectAltName)` section should contain alternate domain/IP addresses of the component.

   1. Creating khulnasoft_web key and csr:
      ```bash
      openssl req -newkey rsa:2048 -nodes -keyout khulnasoft_web.key -subj "/C=US/ST=CA/O=MyOrg/CN=khulnasoft-console-svc" -out khulnasoft_web.csr
      ```

   2. Creating khulnasoft_gateway key and csr:
      ```bash
      openssl req -newkey rsa:2048 -nodes -keyout khulnasoft_gateway.key -subj "/C=US/ST=CA/O=MyOrg/CN=khulnasoft-gateway-svc" -out khulnasoft_gateway.csr
      ```

   here, change the `subjectAltName` accordingly to your domain.

   **4. Verify the CSR's content:**
   - verify the generated component csr content(optional)
      ```shell
      openssl req -in khulnasoft_web.csr -noout -text
      openssl req -in khulnasoft_gateway.csr -noout -text
      ```

   **5. Generate the certificate using the component csr's and key along with the CA Root key:**

   1. Creating khulnasoft_web certificate
      ```shell
      openssl x509 -req \
         -extfile <(printf "subjectAltName=DNS:khulnasoft-console-svc.khulnasoft,DNS:khulnasoft-console-svc,DNS:my-domain.com") \
         -days 365 -in khulnasoft_web.csr \
         -CA rootCA.crt -CAkey rootCA.key -CAcreateserial \
         -out khulnasoft_web.crt
      ```

      here, change the `subjectAltName` accordingly to your domain.

   2. Creating khulnasoft_gateway certificate
      ```shell
      openssl x509 -req \
         -extfile <(printf "subjectAltName=DNS:khulnasoft-gateway-svc.khulnasoft,DNS:khulnasoft-gateway-svc,DNS:my-domain.com") \
         -days 365 -in khulnasoft_gateway.csr \
         -CA rootCA.crt -CAkey rootCA.key -CAcreateserial \
         -out khulnasoft_gateway.crt
      ```

      here, change the `subjectAltName` accordingly to your domain.


   **6. Verify the certificate's content:**
   - verify the generated component certificate content(optional)
      ```shell
      openssl x509 -in khulnasoft_web.crt -text -noout
      openssl x509 -in khulnasoft_gateway.crt -text -noout
      ```

   ### Create secrets with generated certs and change `values.yaml` as mentioned below

   1. Create Kubernetes secrets for server and gateway components using the generated SSL certificates.
      ```shell
      # Example:
      # Change < certificate filenames > respectively
      kubectl create secret generic khulnasoft-web-certs --from-file khulnasoft_web.key --from-file khulnasoft_web.crt --from-file rootCA.crt -n khulnasoft
      kubectl create secret generic khulnasoft-gateway-certs --from-file khulnasoft_gateway.key --from-file khulnasoft_gateway.crt --from-file rootCA.crt -n khulnasoft
      ```

   2. Enable `web.TLS.enable` and `gate.TLS.enable` to `true` in values.yaml
   3. Add the certificates secret name `web.TLS.secretName` and `gateway.TLS.secretName` in values.yaml
   4. Add respective certificate file names to `web.TLS.publicKey_fileName`, `web.TLS.privateKey_fileName`, `web.TLS.rootCA_fileName`(Add rootCA if certs are self-signed), `gate.TLS.publicKey_fileName`, `gate.TLS.privateKey_fileName` and `gate.TLS.rootCA_fileName`(Add rootCA if certs are self-signed) in values.yaml
   5. For enabling mTLS/TLS connection with self-signed or CA certificates between gateway and enforcers please setup mTLS/TLS config for [enforcer chart](../enforcer) and [kube-enforcer](../kube-enforcer/README.md#4-configuring-mtlstls-for-khulnasoft-server-and-khulnasoft-gateway)

## Configure mTLS between server/gateway and external DB
   1. Add the external DB values under `global.db.external` section
   2. Change `global.db.ssl` and `global.db.auditssl` to true
   3. Create secret with external DB public certificate
      ```shell
      kubectl create secret generic <<dbcert_secret_name>> --from-file <<db_certificate.pem_file_path>> -n khulnasoft
      ```
   4. Set the following variables
      ```shell
      global.db.externalDBCerts.enable: true
      global.db.externalDBCerts.certSecretName: <<dbcert_secret_name>>
      ```
   5. Select ssl mode for external databases
      ```shell
      sslmode: require          # accepts: allow | prefer | require | verify-ca | verify-full (Default: Require)
      auditsslmode: require     # accepts: allow | prefer | require | verify-ca | verify-full (Default: Require)
      ```
   6. Note: In Active-Active mode set respective values to pubsub varibales

## Setting active-active mode

   1. Set `activeactive` to true in values.yaml
   2. Also set following configurable variables
      ```shell
      global.db.external.pubsubName
      global.db.external.pubsubHost
      global.db.external.pubsubPort
      global.db.external.pubsubUser
      global.db.external.pubsubPassword
      ```

## Configurable Variables

Parameter | Description                                                                                                                                                                                        | Default                                      | Mandatory
--------- |----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------| ---------
`imageCredentials.create` | Set if to create new pull image secret                                                                                                                                                             | `true`                                       | `YES`
`imageCredentials.name` | Your Docker pull image secret name                                                                                                                                                                 | `khulnasoft-registry-secret`                       | `YES`
`imageCredentials.repositoryUriPrefix` | repository uri prefix for dockerhub set `docker.io`                                                                                                                                                | `registry.khulnasoft.com`                       | `YES`
`imageCredentials.registry` | set the registry url for dockerhub set `index.docker.io/v1/`                                                                                                                                       | `registry.khulnasoft.com`                       | `YES`
`imageCredentials.username` | Your Docker registry (DockerHub, etc.) username                                                                                                                                                    | `khulnasoft-registry-secret`                       | `YES`
`imageCredentials.password` | Your Docker registry (DockerHub, etc.) password                                                                                                                                                    | `unset`                                      | `YES`
`global.platform` | Orchestration platform name (Allowed values are aks, eks, gke, openshift, tkg, tkgi, k8s, rancher, gs, k3s)                                                                                        | `unset`                                      | `YES`
`openshift_route.create` | to create openshift routes for web and gateway                                                                                                                                                     | `false`                                      | `NO`
`rbac.create` | to create default cluster-role and cluster-role bindings according to the platform                                                                                                                 | `true`                                       | `NO`
`serviceAccount.create`    | Enable to true to create serviceaccount                                                                                                                                                            | `true`                                       | `YES`
`serviceAccount.name`      | mention existing service-account name to overwrite the default serviceAccount name, Default is {{ .Release.Namespace }}-sa                                                                         | `unset`                                      | `NO`
`clusterRole.roleRef` | cluster role reference name for cluster rolebinding                                                                                                                                                | `unset`                                      | `NO`
`activeactive` | set for HA Active-Active cluster mode                                                                                                                                                              | `false`                                      
`clustermode` | set for HA Active-Passive cluster mode <br> To be deprecated, use Active-Active instead                                                                                                            | `false`                                      
`admin.createSecret`| Set if to create web secret                                                                                                                                                                     | `unset`                                      | `NO`
`admin.secretName`| Web secret name                                                                                                                                                                                    | `unset`                                      | `NO`
`admin.token`| Use this Khulnasoft license token                                                                                                                                                                        | `unset`                                      | `NO`
`admin.password` | Use this Khulnasoft admin password                                                                                                                                                                       | `unset`                                      | `NO`
`dockerSocket.mount` | boolean parameter if to mount docker socket                                                                                                                                                        | `unset`                                      | `NO`
`dockerSocket.path` | docker socket path                                                                                                                                                                                 | `/var/run/docker.sock`                       | `NO`
`podSecurityPolicy.create` | Enable Pod Security Policies with the required Server capabilities                                                                                                                                 | `false`                                      | `NO`
`podSecurityPolicy.privileged` | Enable privileged permissions to the Server                                                                                                                                                        | `true` if podSecurityPolicy.create is `true` | `NO`
`docker` | Scanning mode direct or docker [link](https://docs.khulnasoft.com/docs/scanning-mode#default-scanning-mode)                                                                                           | `-`                                          | `NO`
`global.db.external.enabled` | Avoid installing a Postgres container and use an external database instead                                                                                                                         | `false`                                      | `YES`
`global.db.external.name` | PostgreSQL DB name                                                                                                                                                                                 | `unset`                                      | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.host` | PostgreSQL DB hostname                                                                                                                                                                             | `unset`                                      | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.port` | PostgreSQL DB port                                                                                                                                                                                 | `unset`                                      | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.user` | PostgreSQL DB username                                                                                                                                                                             | `unset`                                      | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.password` | PostgreSQL DB password                                                                                                                                                                             | `unset`                                      | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.auditName` | PostgreSQL DB audit name                                                                                                                                                                           | `unset`                                      | `NO`
`global.db.external.auditHost` | PostgreSQL DB audit hostname                                                                                                                                                                       | `unset`                                      | `NO`
`global.db.external.auditPort` | PostgreSQL DB audit port                                                                                                                                                                           | `unset`                                      | `NO`
`global.db.external.auditUser` | PostgreSQL DB audit username                                                                                                                                                                       | `unset`                                      | `NO`
`global.db.external.auditPassword` | PostgreSQL DB audit password                                                                                                                                                                       | `unset`                                      | `NO`
`global.db.external.pubsubName` | PostgreSQL DB pubsub name                                                                                                                                                                          | `unset`                                      | `NO`
`global.db.external.pubsubHost` | PostgreSQL DB pubsub hostname                                                                                                                                                                      | `unset`                                      | `NO`
`global.db.external.pubsubPort` | PostgreSQL DB pubsub port                                                                                                                                                                          | `unset`                                      | `NO`
`global.db.external.pubsubUser` | PostgreSQL DB pubsub username                                                                                                                                                                      | `unset`                                      | `NO`
`global.db.external.pubsubPassword` | PostgreSQL DB pubsub password                                                                                                                                                                      | `unset`                                      | `NO`
`global.db.passwordFromSecret.enabled` | Enable to load DB passwords from Secrets                                                                                                                                                           | `false`                                      | `YES`
`global.db.passwordFromSecret.dbPasswordName` | password secret name                                                                                                                                                                               | `null`                                       | `NO`
`global.db.passwordFromSecret.dbPasswordKey` | password secret key                                                                                                                                                                                | `null`                                       | `NO`
`global.db.passwordFromSecret.dbAuditPasswordName` | Audit password secret name                                                                                                                                                                         | `null`                                       | `NO`
`global.db.passwordFromSecret.dbAuditPasswordKey` | Audit password secret key                                                                                                                                                                          | `null`                                       | `NO`
`global.db.passwordFromSecret.dbPubsubPasswordName` | Pubsub password secret name                                                                                                                                                                        | `null`                                       | `NO`
`global.db.passwordFromSecret.dbPubsubPasswordKey` | Pubsub password secret key                                                                                                                                                                         | `null`                                       | `NO`
`global.db.ssl` | If require an SSL-encrypted connection to the Postgres configuration database.                                                                                                                     | 	`false`                                     | `NO`
`global.db.sslmode` | If `ssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require)                                           | `require`                                    | `NO`
`global.db.auditssl` | If require an SSL-encrypted connection to the Postgres configuration audit database.                                                                                                               | 	`false`                                     | `NO`
`global.db.auditsslmode` | If `auditssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require)                                      | `require`                                    | `NO`
`global.db.pubsubssl` | If require an SSL-encrypted connection to the Postgres configuration pubsub database.                                                                                                              | 	`false`                                     | `NO`
`global.db.pubsubsslmode` | If `pubsubssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require)                                     | `require`                                    | `NO`
`global.db.externalDbCerts.enable` | If true, external db can connect with mTLS i.e.., verify-ca/verify-full ssl mode by mounting the ca cert to console & gateway                                                                      | `false`                                      | `NO`
`global.db.externalDbCerts.certSecretName` | If `global.db.externalDbCerts.enable` true, Secret name which holds external db ca certificate files                                                                                               | `null`                                       | `NO`
`global.db.persistence.database.enabled` | If true, Persistent Volume Claim will be created                                                                                                                                                   | 	`true`                                      | `NO`
`global.db.persistence.database.accessModes` | 	Persistent Volume access mode                                                                                                                                                                     | 	`ReadWriteOnce`                             | `NO`
`global.db.persistence.database.size` | 	Persistent Volume size                                                                                                                                                                            | `30Gi`                                       | `NO`
`global.db.persistence.database.storageClass` | 	Persistent Volume Storage Class                                                                                                                                                                   | `unset`                                      | `NO`
`global.db.persistence.audit_database.enabled` | If true, Persistent Volume Claim will be created for audit database                                                                                                                                | 	`true`                                      | `NO`
`global.db.persistence.audit_database.accessModes` | 	Persistent Volume access mode for audit database                                                                                                                                                  | 	`ReadWriteOnce`                             | `NO`
`global.db.persistence.audit_database.size` | 	Persistent Volume size for audit database                                                                                                                                                         | `30Gi`                                       | `NO`
`global.db.persistence.audit_database.storageClass` | 	Persistent Volume Storage Class for audit database                                                                                                                                                | `unset`                                      | `NO`
`global.db.env_size` | Set this to tune DB parameters                                                                                                                                                                     | `S`                                          | `YES`</br >`Possible values: “S” (default), “M”, “L”`
`global.db.image.repository` | the docker image name to use                                                                                                                                                                       | `database`                                   | `NO`
`global.db.image.tag` | The image tag to use.                                                                                                                                                                              | `2022.4`                                     | `NO`
`global.db.image.pullPolicy` | The kubernetes image pull policy.                                                                                                                                                                  | `IfNotPresent`                               | `NO`
`global.db.service.type` | k8s service type                                                                                                                                                                                   | `ClusterIP`                                  | `NO`
`global.db.resources` | 	Resource requests and limits                                                                                                                                                                      | `{}`                                         | `NO`
`global.db.nodeSelector` | 	Kubernetes node selector                                                                                                                                                                          | `{}`                                         | `NO`
`global.db.tolerations` | 	Kubernetes node tolerations	                                                                                                                                                                      | `[]`                                         | `NO`
`global.db.affinity` | 	Kubernetes node affinity                                                                                                                                                                          | `{}`                                         | `NO`
`global.db.podAnnotations` | Kubernetes pod annotations                                                                                                                                                                         | `{}`                                         | `NO`
`global.db.securityContext` | Set of security context for the container                                                                                                                                                          | `nil`                                        | `NO`
`global.db.extraEnvironmentVars` | is a list of extra environment variables to set in the database deployments.                                                                                                                       | `{}`                                         | `NO`
`global.db.extraSecretEnvironmentVars` | is a list of extra environment variables to set in the database deployments, these variables take value from existing Secret objects.                                                              | `[]`                                         | `NO`
`gateway.image.repository` | the docker image name to use                                                                                                                                                                       | `gateway`                                    | `NO`
`gateway.enabled` | Deploy gateway chart with server chart                                                                                                                                                             | `True`                                       | `NO`
`gateway.image.tag` | The image tag to use.                                                                                                                                                                              | `2022.4`                                     | `NO`
`gateway.image.pullPolicy` | The kubernetes image pull policy.                                                                                                                                                                  | `IfNotPresent`                               | `NO`
`gateway.service.type` | k8s service type                                                                                                                                                                                   | `ClusterIP`                                  | `NO`
`gateway.service.loadbalancerIP` | can specify loadBalancerIP address for khulnasoft-web in AKS platform                                                                                                                                    | `null`                                       | `NO`
`gateway.service.annotations` | 	service annotations	                                                                                                                                                                              | `{}`                                         | `NO`
`gateway.service.ports` | array of ports settings                                                                                                                                                                            | `array`                                      | `NO`
`gateway.service.loadBalancerSourceRanges` | In order to limit which client IP's can access the Network Load Balancer, specify a list of CIDRs                                                                                                  | `null`                                       | `NO`
`gateway.serviceAccount.name` | The name of the SA to deploy the gateway                                                                                                                                                           | `khulnasoft-sa`                                    | `NO`
`gateway.serviceAccount.annotations` | The annotations of the SA to deploy the gateway                                                                                                                                                    | ``                                    | `NO`
`gateway.publicIP` | gateway public ip                                                                                                                                                                                  | `khulnasoft-gateway`                               | `NO`
`gateway.replicaCount` | replica count                                                                                                                                                                                      | `1`                                          | `NO`
`gateway.resources` | 	Resource requests and limits                                                                                                                                                                      | `{}`                                         | `NO`
`gateway.nodeSelector` | 	Kubernetes node selector                                                                                                                                                                          | `{}`                                         | `NO`
`gateway.tolerations` | 	Kubernetes node tolerations	                                                                                                                                                                      | `[]`                                         | `NO`
`gateway.affinity` | 	Kubernetes node affinity                                                                                                                                                                          | `{}`                                         | `NO`
`gateway.podAnnotations` | Kubernetes pod annotations                                                                                                                                                                         | `{}`                                         | `NO`
`gateway.securityContext` | Set of security context for the container                                                                                                                                                          | `nil`                                        | `NO`
`gateway.pdb.minAvailable` | Set minimum available value for gate pod PDB                                                                                                                                                       | `1`                                          | `NO`
`gateway.TLS.enabled` | If require secure channel communication                                                                                                                                                            | `false`                                      | `NO`
`gateway.TLS.secretName` | certificates secret name                                                                                                                                                                           | `nil`                                        | `YES` <br /> `if gate.TLS.enabled is set to true`
`gateway.TLS.publicKey_fileName` | filename of the public key eg: khulnasoft_gateway.crt                                                                                                                                                    | `nil`                                        |  `YES` <br /> `if gate.TLS.enabled is set to true`
`gateway.TLS.privateKey_fileName`   | filename of the private key eg: khulnasoft_gateway.key                                                                                                                                                   | `nil`                                        |  `YES` <br /> `if gate.TLS.enabled is set to true`
`gateway.TLS.rootCA_fileName` | filename of the rootCA, if using self-signed certificates eg: rootCA.crt                                                                                                                           | `nil`                                        |  `NO` <br /> `if gate.TLS.enabled is set to true and using self-signed certificates for TLS/mTLS`
`gateway.TLS.khulnasoft_verify_enforcer` | change it to "1" or "0" for enabling/disabling mTLS between enforcer and gateway/envoy                                                                                                             | `0`                                          |  `YES` <br /> `if gate.TLS.enabled is set to true`
`gateway.extraEnvironmentVars` | is a list of extra environment variables to set in the gateway deployments.                                                                                                                        | `{}`                                         | `NO`
`gateway.extraSecretEnvironmentVars` | is a list of extra environment variables to set in the gateway deployments, these variables take value from existing Secret objects.                                                               | `[]`                                         | `NO`
`gateway.headlessService` | create headless service for envoy                                                                                                                                                                  | `true`                                       | `NO`
`web.image.repository` | the docker image name to use                                                                                                                                                                       | `console`                                    | `NO`
`web.image.tag` | The image tag to use.                                                                                                                                                                              | `2022.4`                                     | `NO`
`web.image.pullPolicy` | The kubernetes image pull policy.                                                                                                                                                                  | `IfNotPresent`                               | `NO`
`web.service.type` | k8s service type                                                                                                                                                                                   | `LoadBalancer`                               | `NO`
`web.service.loadbalancerIP` | can specify loadBalancerIP address for khulnasoft-web in AKS platform                                                                                                                                    | `null`                                       | `NO`
`web.service.annotations` | 	service annotations	                                                                                                                                                                              | `{}`                                         | `NO`
`web.service.ports` | array of ports settings                                                                                                                                                                            | `array`                                      | `NO`
`web.replicaCount` | replica count                                                                                                                                                                                      | `1`                                          | `NO`
`web.resources` | 	Resource requests and limits                                                                                                                                                                      | `{}`                                         | `NO`
`web.nodeSelector` | 	Kubernetes node selector                                                                                                                                                                          | `{}`                                         | `NO`
`web.tolerations` | 	Kubernetes node tolerations	                                                                                                                                                                      | `[]`                                         | `NO`
`web.affinity` | 	Kubernetes node affinity                                                                                                                                                                          | `{}`                                         | `NO`
`web.podAnnotations` | Kubernetes pod annotations                                                                                                                                                                         | `{}`                                         | `NO`
`web.ingress.enabled` | 	If true, Ingress will be created                                                                                                                                                                  | `false`                                      | `NO`
`web.ingress.apiVersion` | 	Override the API version of the Ingress                                                                                                                                                           | `false`                                      | `NO`
`web.ingress.annotations` | 	Ingress annotations	                                                                                                                                                                              | `[]`                                         | `NO`
`web.ingress.hosts` | Ingress hostnames                                                                                                                                                                                  | 	`[]`                                        | `NO`
`web.ingress.path` | 	Ingress Path                                                                                                                                                                                      | `/`                                          | `NO`
`web.ingress.tls` | 	Ingress TLS configuration (YAML)                                                                                                                                                                  | `[]`                                         | `NO`
`web.securityContext` | Set of security context for the container                                                                                                                                                          | `nil`                                        | `NO`
`web.pdb.minAvailable` | Set minimum available value for web pod PDB                                                                                                                                                        | `1`                                          | `NO`
`web.TLS.enabled` | If require secure channel communication                                                                                                                                                            | `false`                                      | `NO`
`web.TLS.secretName` | certificates secret name                                                                                                                                                                           | `nil`                                        | `NO`
`web.TLS.publicKey_fileName` | filename of the public key eg: khulnasoft_web.crt                                                                                                                                                        | `nil`                                        |  `YES` <br /> `if gate.TLS.enabled is set to true`
`web.TLS.privateKey_fileName`   | filename of the private key eg: khulnasoft_web.key                                                                                                                                                       | `nil`                                        |  `YES` <br /> `if gate.TLS.enabled is set to true`
`web.TLS.rootCA_fileName` | filename of the rootCA, if using self-signed certificates eg: rootCA.crt                                                                                                                           | `nil`                                        |  `NO` <br /> `if gate.TLS.enabled is set to true and using self-signed certificates for TLS/mTLS`
`web.maintenance_db.name` | If Configured to use custom maintenance DB specify the DB name                                                                                                                                     | `unset`                                      | `NO`
`web.extraEnvironmentVars` | is a list of extra environment variables to set in the web deployments.                                                                                                                            | `{}`                                         | `NO`
`web.extraSecretEnvironmentVars` | is a list of extra environment variables to set in the web deployments, these variables take value from existing Secret objects.                                                                   | `[]`                                         | `NO`
`envoy.enabled` | enabled envoy deployment.                                                                                                                                                                          | `false`                                      | `NO`
`envoy.replicaCount` | replica count                                                                                                                                                                                      | `1`                                          | `NO`
`envoy.image.repository` | the docker image name to use                                                                                                                                                                       | `envoyproxy/envoy-alpine`                    | `NO`
`envoy.image.tag` | The image tag to use.                                                                                                                                                                              | `v1.14.1`                                    | `NO`
`envoy.image.pullPolicy` | The kubernetes image pull policy.                                                                                                                                                                  | `IfNotPresent`                               | `NO`
`envoy.service.type` | k8s service type                                                                                                                                                                                   | `LoadBalancer`                               | `NO`
`envoy.service.loadbalancerIP` | can specify loadBalancerIP address for khulnasoft-web in AKS platform                                                                                                                                    | `null`                                       | `NO`
`envoy.service.annotations` | use this field to pass additional annotations for the service, useful to drive Cloud providers behaviour in creating the LB resource. E.g. `service.beta.kubernetes.io/aws-load-balancer-type: nlb` | `{}`                                         | `NO`
`envoy.service.ports` | array of ports settings                                                                                                                                                                            | `array`                                      | `NO`
`envoy.pdb.minAvailable` | Set minimum available value for envoy pod PDB                                                                                                                                                      | `1`                                          | `NO`
`envoy.TLS.listener.enabled` | enable to load custom self-signed or CA certs                                                                                                                                                      | `false`                                      | `NO` <br /> `if envoy.enabled is set to true`
`envoy.TLS.listener.secretName` | certificates secret name                                                                                                                                                                           | `nil`                                        | `NO` <br /> `if envoy.enabled is set to true`
`envoy.TLS.listener.publicKey_fileName` | filename of the public key eg: khulnasoft-lb.fqdn.crt                                                                                                                                                    | `nil`                                        |  `YES` <br /> `if envoy.enabled is set to true`
`envoy.TLS.listener.privateKey_fileName`   | filename of the private key eg: khulnasoft-lb.fqdn.key                                                                                                                                                   | `nil`                                        |  `YES` <br /> `if envoy.enabled is set to true`
`envoy.TLS.listener.rootCA_fileName` | filename of the rootCA, if using self-signed certificates eg: rootCA.crt                                                                                                                           | `nil`                                        |  `NO`
`envoy.TLS.cluster.enabled` | If require secure channel communication between Envoy and Gateway                                                                                                                                  | `false`                                      | `NO`
`envoy.TLS.cluster.secretName` | certificates secret name                                                                                                                                                                           | `nil`                                        | `NO`
`envoy.TLS.cluster.publicKey_fileName` | filename of the public key eg: khulnasoft-lb.crt                                                                                                                                                         | `nil`                                        |  `NO`
`envoy.TLS.cluster.privateKey_fileName`   | filename of the private key eg: khulnasoft-lb.key                                                                                                                                                        | `nil`                                        |  `NO`
`envoy.TLS.cluster.rootCA_fileName` | filename of the rootCA, if using self-signed certificates eg: rootCA.crt                                                                                                                           | `nil`                                        |  `NO`
`envoy.livenessProbe` | liveness probes configuration for envoy                                                                                                                                                            | `{}`                                         | `NO`
`envoy.readinessProbe` | readiness probes configuration for envoy                                                                                                                                                           | `{}`                                         | `NO`
`envoy.resources` | 	Resource requests and limits                                                                                                                                                                      | `{}`                                         | `NO`
`envoy.nodeSelector` | 	Kubernetes node selector                                                                                                                                                                          | `{}`                                         | `NO`
`envoy.tolerations` | 	Kubernetes node tolerations	                                                                                                                                                                      | `[]`                                         | `NO`
`envoy.podAnnotations` | Kubernetes pod annotations                                                                                                                                                                         | `{}`                                         | `NO`
`envoy.affinity` | 	Kubernetes node affinity                                                                                                                                                                          | `{}`                                         | `NO`
`envoy.securityContext` | Set of security context for the container                                                                                                                                                          | `nil`                                        | `NO`
`envoy.files.envoy.yaml` | content of a full envoy configuration file as documented in https://www.envoyproxy.io/docs/envoy/latest/configuration/configuration                                                                | check [values.yaml](values.yaml)             
`initContainers.enabled` | If true, init container definitions will take effect                                                                                                                                               | `Fasle`                                      | `NO`
`initContainers.ContainersDefinitions` | Container definitions for init containers                                                                                                                                                          | `nil`                                        | `NO`
`sidecarContainers.enabled` | If true, init sidecar containers definitions will take effect                                                                                                                                      | `Fasle`                                      | `NO`
`sidecarContainers.ContainersDefinitions` | Container definitions for sidecar containers                                                                                                                                                       | `nil`                                        | `NO`

## Issues and feedback

If you encounter any problems or would like to give us feedback on deployments, we encourage you to raise issues here on GitHub.
