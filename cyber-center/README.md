<img src="https://avatars3.githubusercontent.com/u/12783832?s=200&v=4" height="100" width="100" /><img src="https://avatars3.githubusercontent.com/u/15859888?s=200&v=4" width="100" height="100"/>

# Khulnasoft Security Cyber Center Helm Chart

These are Helm charts for installation and maintenance of Khulnasoft Container Security Cyber-Center

## Contents

- [Khulnasoft Security Cyber Center Helm Chart](#khulnasoft-security-cyber-center-helm-chart)
  - [Contents](#contents)
  - [Prerequisites](#prerequisites)
    - [Container Registry Credentials](#container-registry-credentials)
  - [Installing the Chart](#installing-the-chart)
    - [Installing Khulnasoft Cyber-Center from Github Repo](#installing-khulnasoft-cyber-center-from-github-repo)
    - [Installing Khulnasoft Cyber-Center from Helm Private Repository](#installing-khulnasoft-cyber-center-from-helm-private-repository)
  - [Configuring mTLS/TLS](#configuring-mtlstls)
  - [Guide how to connect to offline cyber-center from khulnasoft console](#guide-how-to-connect-to-offline-cyber-center-from-khulnasoft-console)
  - [Configurable Variables](#configurable-variables)
    - [Cyber-Center](#cyber-center)
  - [Issues and feedback](#issues-and-feedback)

## Prerequisites

### Container Registry Credentials

[Link](../docs/imagepullsecret.md)

## Installing the Chart
Follow the steps in this section for production grade deployments. You can either clone khulnasoft-helm git repo or you can add our helm private repository ([https://helm.khulnasoft.com](https://helm.khulnasoft.com))

### Installing Khulnasoft Cyber-Center from Helm Private Repository

* Add Khulnasoft Helm Repository
```shell
helm repo add khulnasoft-helm https://helm.khulnasoft.com
helm repo update
```

* Check for available chart versions either from [Changelog](./CHANGELOG.md) or by running the below command
```shell
helm search repo khulnasoft-helm/cyber-center --versions
```

* Install Khulnasoft Cyber-Center

```shell
helm upgrade --install --namespace khulnasoft khulnasoft-cyber-center khulnasoft-helm/cyber-center --set imageCredentials.username=<>,imageCredentials.password=<>
```


## Configuring mTLS/TLS

In order to support L7 / gRPC communication between cyber-center and server or cyber-center and scanner it is recommended to follow the detailed steps to enable and deploy a cyber-center.

   1. The Cyber-Center should connect to the Envoy/Gateway service and verify its certificate. If Envoy/Gateway certificate is signed by a public provider (e.g., Let’s Encrypt), the Cyber-Center will be able to verify the certificate without being given a root certificate. Otherwise, Envoy/Gateway root certificate should be accessible to the Cyber-Center from a environment variable path that is mounted at /opt/khulnasoft/ssl/

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
      openssl genrsa -out khulnasoft_cc_mydomain.com.key 2048
      # Create the signing (csr)
      openssl req -new -key khulnasoft_cc_mydomain.com.key -out khulnasoft_cc_mydomain.com.csr
      # Verify the csr content
      openssl req -in khulnasoft_cc_mydomain.com.csr -noout -text
      #####################################################################################
      # Generate the certificate using the khulnasoft_cc_mydomain csr and key along with the CA Root key
      #####################################################################################
      openssl x509 -req -in khulnasoft_cc_mydomain.com.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out khulnasoft_cc_mydomain.com.crt -days 500 -sha256
      #####################################################################################

      # If you wish to use a Public CA like GoDaddy or LetsEncrypt please
      # submit the khulnasoft_cc_mydomain csr to the respective CA to generate khulnasoft_cc_mydomain crt
      ```

   2. Create cyber-center agent cert secret

      ```shell
      ## Example: 
      ## Change < certificate filenames > respectively
      kubectl create secret generic khulnasoft-cc-certs --from-file <khulnasoft_cyber-center_private.key> --from-file <khulnasoft_cyber-center_public.crt> --from-file <rootCA.crt> -n khulnasoft
      ```

   3. Enable `TLS.enable`  to `true` in values.yaml
   4. Add the certificates secret name `TLS.secretName` in values.yaml
   5. Add respective certificate file names to `TLS.publicKey_fileName`, `TLS.privateKey_fileName` and `TLS.rootCA_fileName`(Add rootCA if certs are self-signed) in values.yaml


## Guide how to connect to offline cyber-center from khulnasoft console

Please login into Khulnasoft Web UI then go to `Khulnasoft CyberCenters` section under `Settings` tab to connect to a new offline cyber-center. 

1. Change the Address to offline cybercenter address `eg: http://khulnasoft-cc:443`
2. Test the connection once it is success save the changes.
3. Disable `Fast Scanning` under `Scanning` section in `Setting` tab.

For more information please check [Link](https://docs.khulnasoft.com/docs/cybercenter-configuration)

## Configurable Variables

### Cyber-Center

Parameter | Description | Default                | Mandatory 
--------- | ----------- |------------------------| ------- 
`imageCredentials.create` | Set if to create new pull image secret | `false`                | `YES - New cluster`
`imageCredentials.name` | Your Docker pull image secret name | `khulnasoft-registry-secret` | `YES - New cluster`
`imageCredentials.repositoryUriPrefix` | repository uri prefix for dockerhub set `docker.io` | `registry.khulnasoft.com` | `YES - New cluster`
`imageCredentials.registry` | set the registry url for dockerhub set `index.docker.io/v1/` | `registry.khulnasoft.com` | `YES - New cluster`
`imageCredentials.username` | Your Docker registry (DockerHub, etc.) username | `khulnasoft-registry-secret` | `YES - New cluster`
`imageCredentials.password` | Your Docker registry (DockerHub, etc.) password | `unset`                | `YES - New cluster`
`serviceAccount.create` | enable to create khulnasoft-sa serviceaccount if it is missing in the environment | `false`                | `YES - New cluster`
`serviceAccount.name` | service acccount name | `khulnasoft-sa`              | `NO`
`image.repository` | the docker image name to use | `cc-standard`          | `YES`
`image.tag` | The image tag to use. | `2022.4`               | `YES`
`image.pullPolicy` | The kubernetes image pull policy. | `Always`               | `NO`
`service.type` | k8s service type | `ClusterIP`            | `NO`
`service.annotations` |	service annotations	| `{}`                   | `NO`
`service.ports` | array of ports settings | `array`                | `NO`
`tolerations` |	Kubernetes node tolerations	| `[]`                   | `NO`
`podAnnotations` | Kubernetes pod annotations | `{}`                   | `NO`
`resources` |	Resource requests and limits | `{}`                   | `NO`
`nodeSelector` |	Kubernetes node selector	| `{}`                   | `NO`
`affinity` |	Kubernetes node affinity | `{}`                   | `NO`
`TLS.enabled` | If require secure channel communication | `false`                | `NO`
`TLS.secretName` | certificates secret name | `nil`                  | `YES` <br /> `if TLS.enabled is set to true`
`TLS.publicKey_fileName` | filename of the public key eg: khulnasoft_cyber-center.crt | `nil`                  |  `YES` <br /> `if TLS.enabled is set to true`
`TLS.privateKey_fileName`   | filename of the private key eg: khulnasoft_cyber-center.key | `nil`                  |  `YES` <br /> `if TLS.enabled is set to true`
`TLS.rootCA_fileName` |  filename of the rootCA, if using self-signed certificates eg: rootCA.crt | `nil`                  |  `NO` <br /> `if TLS.enabled is set to true and using self-signed certificates for TLS/mTLS`


> Note: that `imageCredentials.create` is false and if you need to create image pull secret please update to true, set the username and password for the registry and `serviceAccount.create` is false and if you're environment is new or not having khulnasoft-sa serviceaccount please update it to true.

## Issues and feedback

If you encounter any problems or would like to give us feedback on deployments, we encourage you to raise issues here on GitHub.
