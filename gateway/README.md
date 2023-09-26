<img src="https://avatars3.githubusercontent.com/u/12783832?s=200&v=4" height="100" width="100" /><img src="https://avatars3.githubusercontent.com/u/15859888?s=200&v=4" width="100" height="100"/>

# Khulnasoft Security Gateway Helm Chart

Helm chart for installation and maintenance of Khulnasoft Container Security Platform Gateway component to support multi-cluster use-case.

## Contents

- [Khulnasoft Security Gateway Helm Chart](#khulnasoft-security-gateway-helm-chart)
  - [Contents](#contents)
  - [Prerequisites](#prerequisites)
    - [Container Registry Credentials](#container-registry-credentials)
  - [Installing the Chart](#installing-the-chart)
    - [Installing Khulnasoft Gateway from Github Repo](#installing-khulnasoft-gateway-from-github-repo)
        - [Edit values.yaml file in gateway directory with following details as prerequisite](#edit-valuesyaml-file-in-gateway-directory-with-following-details-as-prerequisite)
    - [Installing Khulnasoft Gateway from Helm Private Repository](#installing-khulnasoft-gateway-from-helm-private-repository)
  - [Configure mTLS between server/gateway and external DB](#configure-mtls-between-servergateway-and-external-db)
  - [Configurable Variables](#configurable-variables)

## Prerequisites

### Container Registry Credentials

[Link](../docs/imagepullsecret.md)


## Installing the Chart
Follow the steps in this section for production grade deployments. You can either clone khulnasoft-helm git repo or you can add our helm private repository ([https://helm.khulnasoft.com](https://helm.khulnasoft.com))

### Installing Khulnasoft Gateway from Github Repo

1. Clone the GitHub repository with the charts
    ```shell
    git clone -b 2022.4 https://github.com/khulnasoft-lab/khulnasoft-helm.git
    cd khulnasoft-helm/
    ```

##### Edit values.yaml file in gateway directory with following details as prerequisite
```json
  1. global.db.external.name - External database name
  2. global.db.external.host - External database public IP or endpoint URL
  3. global.db.external.port - External database port
  4. global.db.external.user - External database username
  5. global.db.external.password - External database password
  6. global.db.external.auditName - External audit database name
  7. global.db.external.auditHost - External audit database public IP or endpoint URL
  8. global.db.external.auditPort - External audit database port
  9. global.db.external.auditUser - External audit database username
  10. global.db.external.auditPassword - External audit database password
  11. console.publicIP - Khulnasoft CSP console public IP or endpoint URL
  12. console.publicPort - Khulnasoft CSP console public port
  // To load DB passswords from secrets:
  // change passwordFromSecret.enable to true
  // for loading DB password from secret edit passwordFromSecret.dbPasswordName  passwordFromSecret.dbPasswordName with secret name and secret key
  // for loading Audit-DB password from secret edit passwordFromSecret.dbAuditPasswordName  passwordFromSecret.dbAuditPasswordName with secret name and secret key
  // for loading PubSub DB password from secret edit passwordFromSecret.dbPubsubPasswordName  passwordFromSecret.dbPubsubPasswordName with secret name and secret key
```

2. Create `khulnasoft` namespace if gateway is being deployed to a new cluster.
    ```shell
    kubectl create namespace khulnasoft
    ```

3. Install Khulnasoft
    ```shell
    helm upgrade --install --namespace khulnasoft gateway ./gateway --set imageCredentials.username=<>,imageCredentials.password=<>,platform=<>
    ```

### Installing Khulnasoft Gateway from Helm Private Repository

1. Add Khulnasoft Helm Repository
    ```shell
    helm repo add khulnasoft-helm https://helm.khulnasoft.com
    ```

2. Check for available chart versions either from [Changelog](./CHANGELOG.md) or by running the below command
    ```shell
    helm search repo khulnasoft-helm/gateway --versions
    ```

3. Create `khulnasoft` namespace if gateway is being deployed to a new cluster.
    ```shell
    kubectl create namespace khulnasoft
    ```

4. copy the values.yaml content from  [Values.yaml](./values.yaml) to a file and make the respective changes [values](#edit-valuesyaml-file-in-gateway-directory-with-following-details-as-prerequisite) then run the following command:
    ```shell
    helm upgrade --install --namespace khulnasoft gateway khulnasoft-helm/gateway --values values.yaml --version <>
    ```

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

## Configurable Variables

Parameter | Description                                                                                                                                                    | Default             | Mandatory |
--------- |----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------| --------- |
`imageCredentials.create`| Set if to create new pull image secret                                                                                                                         | `true`              | `YES`
`imageCredentials.name` | Your Docker pull image secret name                                                                                                                             | `khulnasoft-registry-secret` | `YES`
`imageCredentials.repositoryUriPrefix` | repository uri prefix for dockerhub set `docker.io`                                                                                                            | `registry.khulnasoft.com` | `YES`
`imageCredentials.registry` | set the registry url for dockerhub set `index.docker.io/v1/`                                                                                                   | `registry.khulnasoft.com` | `YES`
`imageCredentials.username` | Your Docker registry (DockerHub, etc.) username                                                                                                                | `khulnasoft-registry-secret` | `YES`
`imageCredentials.password` | Your Docker registry (DockerHub, etc.) password                                                                                                                | `unset`             | `YES`
`rbac.create` | To create RBAC for gateway component                                                                                                                           | `false`             | `YES` <br />`if gateway alone is deploying in new cluster` 
`clusterRole.roleRef` | cluster role reference name for cluster rolebinding                                                                                                            | `unset`             | `NO`
`console.publicIP` | Address of the server                                                                                                                                          | `khulnasoft-console-svc`  | `YES`
`console.publicPort` | Server endpoint Port value                                                                                                                                     | `443`               | `YES`
`serviceAccount.name` | name of the ServiceAccount                                                                                                                                     | `khulnasoft-sa`           | `YES`
`serviceAccount.annotations` | Annotations of the ServiceAccount                                                                                                                              | ``              | `NO`
`headlessService` | create headless service for envoy                                                                                                                              | `true`              | `YES`
`global.platform` | Orchestration platform name (Allowed values are aks, eks, gke, openshift, tkg, tkgi, k8s, rancher, gs, k3s)                                                    | `unset`             | `YES`
`global.db.external.enabled` | Avoid installing a Postgres container and use an external database instead                                                                                     | `false`             | `YES`
`global.db.external.name` | PostgreSQL DB name                                                                                                                                             | `unset`             | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.host` | PostgreSQL DB hostname                                                                                                                                         | `unset`             | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.port` | PostgreSQL DB port                                                                                                                                             | `unset`             | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.user` | PostgreSQL DB username                                                                                                                                         | `unset`             | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.password` | PostgreSQL DB password                                                                                                                                         | `unset`             | `YES`<br />`if db.external.enabled is set to true`
`global.db.external.auditName` | PostgreSQL DB audit name                                                                                                                                       | `unset`             | `NO`
`global.db.external.auditHost` | PostgreSQL DB audit hostname                                                                                                                                   | `unset`             | `NO`
`global.db.external.auditPort` | PostgreSQL DB audit port                                                                                                                                       | `unset`             | `NO`
`global.db.external.auditUser` | PostgreSQL DB audit username                                                                                                                                   | `unset`             | `NO`
`global.db.external.auditPassword` | PostgreSQL DB audit password                                                                                                                                   | `unset`             | `NO`
`global.db.external.pubsubName` | PostgreSQL DB pubsub name                                                                                                                                      | `unset`             | `NO`
`global.db.external.pubsubHost` | PostgreSQL DB pubsub hostname                                                                                                                                  | `unset`             | `NO`
`global.db.external.pubsubPort` | PostgreSQL DB pubsub port                                                                                                                                      | `unset`             | `NO`
`global.db.external.pubsubUser` | PostgreSQL DB pubsub username                                                                                                                                  | `unset`             | `NO`
`global.db.external.pubsubPassword` | PostgreSQL DB pubsub password                                                                                                                                  | `unset`             | `NO`
`global.db.passwordFromSecret.enabled` | Enable to load DB passwords from Secrets                                                                                                                       | `false`             | `YES`
`global.db.passwordFromSecret.dbPasswordName` | password secret name                                                                                                                                           | `null`              | `NO`
`global.db.passwordFromSecret.dbPasswordKey` | password secret key                                                                                                                                            | `null`              | `NO`
`global.db.passwordFromSecret.dbAuditPasswordName` | Audit password secret name                                                                                                                                     | `null`              | `NO`
`global.db.passwordFromSecret.dbAuditPasswordKey` | Audit password secret key                                                                                                                                      | `null`              | `NO`
`global.db.passwordFromSecret.dbPubsubPasswordName` | Pubsub password secret name                                                                                                                                    | `null`              | `NO`
`global.db.passwordFromSecret.dbPubsubPasswordKey` | Pubsub password secret key                                                                                                                                     | `null`              | `NO`
`global.db.ssl` | If require an SSL-encrypted connection to the Postgres configuration database.                                                                                 | 	`false`      | `NO`
`global.db.sslmode` | If `ssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require)       | `require` | `NO`
`global.db.auditssl` | If require an SSL-encrypted connection to the Postgres configuration audit database.                                                                           | 	`false`      | `NO`
`global.db.auditsslmode` | If `auditssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require)  | `require` | `NO`
`global.db.pubsubssl` | If require an SSL-encrypted connection to the Postgres configuration pubsub database.                                                                          | 	`false`      | `NO`
`global.db.pubsubsslmode` | If `pubsubssl` is true, select the type of SSL mode between db and console/gateway  accepts: allow, prefer, require, verify-ca, verify-full (Default: Require) | `require` | `NO`
`global.db.externalDbCerts.enable` | If true, external db can connect with mTLS i.e.., verify-ca/verify-full ssl mode by mounting the ca cert to console & gateway                                  | `false` | `NO`
`global.db.externalDbCerts.certSecretName` | If `global.db.externalDbCerts.enable` true, Secret name which holds external db ca certificate files                                                           | `null` | `NO`
`image.repository` | the docker image name to use                                                                                                                                   | `gateway`           | `NO`
`image.tag` | The image tag to use.                                                                                                                                          | `2022.4`            | `NO`
`image.pullPolicy` | The kubernetes image pull policy.                                                                                                                              | `IfNotPresent`      | `NO`
`service.type` | k8s service type                                                                                                                                               | `ClusterIP`         | `NO`
`service.loadbalancerIP` | can specify loadBalancerIP address for khulnasoft-gateway in AKS platform                                                                                            | `null`              | `NO`
`service.loadBalancerSourceRanges` | In order to limit which client IP's can access the Network Load Balancer, specify a list of CIDRs                                                              | `null`              | `NO`
`service.annotations` | 	service annotations	                                                                                                                                          | `{}`                | `NO`
`service.ports` | array of ports settings                                                                                                                                        | `array`             | `NO`
`publicIP` | gateway public ip                                                                                                                                              | `khulnasoft-gateway`      | `NO`
`replicaCount` | replica count                                                                                                                                                  | `1`                 | `NO`
`resources` | 	Resource requests and limits                                                                                                                                  | `{}`                | `NO`
`nodeSelector` | 	Kubernetes node selector	                                                                                                                                     | `{}`                | `NO`
`tolerations` | 	Kubernetes node tolerations	                                                                                                                                  | `[]`                | `NO`
`affinity` | 	Kubernetes node affinity                                                                                                                                      | `{}`                | `NO`
`podAnnotations` | Kubernetes pod annotations                                                                                                                                     | `{}`                | `NO`
`securityContext` | Set of security context for the container                                                                                                                      | `nil`               | `NO`
`pdb.minAvailable` | Set minimum available value for gate pod PDB                                                                                                                   | `1`                 | `NO`
`TLS.enabled` | If require secure channel communication                                                                                                                        | `false`             | `NO`
`TLS.secretName` | certificates secret name                                                                                                                                       | `nil`               | `YES` <br /> `if gate.TLS.enabled is set to true`
`TLS.publicKey_fileName` | filename of the public key eg: khulnasoft_gateway.crt                                                                                                                | `nil`               |  `YES` <br /> `if gate.TLS.enabled is set to true`
`TLS.privateKey_fileName`   | filename of the private key eg: khulnasoft_gateway.key                                                                                                               | `nil`               |  `YES` <br /> `if gate.TLS.enabled is set to true`
`TLS.rootCA_fileName` | filename of the rootCA, if using self-signed certificates eg: rootCA.crt                                                                                       | `nil`               |  `NO` <br /> `if gate.TLS.enabled is set to true and using self-signed certificates for TLS/mTLS`
`TLS.khulnasoft_verify_enforcer` | change it to "1" or "0" for enabling/disabling mTLS between enforcer and gateway/envoy                                                                         | `0`                 |  `YES` <br /> `if gate.TLS.enabled is set to true`
`extraEnvironmentVars` | is a list of extra environment variables to set in the gateway deployments.                                                                                    | `{}`                | `NO`
`extraSecretEnvironmentVars` | is a list of extra environment variables to set in the gateway deployments, these variables take value from existing Secret objects.                           | `[]`                | `NO`
