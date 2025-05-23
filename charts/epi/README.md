# epi

![Version: 0.7.13](https://img.shields.io/badge/Version-0.7.13-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 4.0.0](https://img.shields.io/badge/AppVersion-4.0.0-informational?style=flat-square)

A Helm chart for Pharma Ledger epi (electronic product information) application

## Requirements

- [helm 3](https://helm.sh/docs/intro/install/)
- These mandatory configuration values:
  - Domain - The Domain - e.g. `epipoc`
  - Sub Domain - The Sub Domain - e.g. `epipoc.my-company`
  - Vault Domain - The Vault Domain - e.g. `vault.my-company`
  - ethadapterUrl - The Full URL of the Ethadapter including protocol and port -  e.g. "https://ethadapter.my-company.com:3000"
  - bdnsHosts - The Centrally managed and provided BDNS Hosts Config
  - buildSecretKey - A secret pass phrase for de/encrpytion of the initially generated private keys of the wallets.

## Usage

- [Here](./README.md#values) is a full list of all configuration values.
- The [values.yaml file](./values.yaml) shows the raw view of all configuration values.

## Changelog


- From 0.7.12 to 0.7.13 - Defaults to use epi 4.0.0
  - Change on builder to run the migrations


- On 0.7.12 - Defaults to use epi 4.0.0
  - Changes was done on apihub.json to add couchdb
  - Created an independent apihub.json for leaflet-reader and added to secret
  - Added affinity rules
  - Added one more ingress route - /metadata/*

- From 0.3.x to 0.4.x - Defaults to use epi v1.3.1
  - Removed Seedsbackup (required for epi <= 1.2.0)
  - `values.yaml` has significant and breaking changes
  - Increased security by design: Run as non-root user, do not allow privilegeEscalation, remove all capabilites from container
  - Sensitive configuration data is stored in Kubernetes Secret instead of ConfigMaps.
  - Support for secret injection via *CSI Secrets Driver* instead of using Kubernetes Secret.
  - Option to use an existing PersistentVolumeClaim instead of creating a new one.

- From 0.2.x to 0.3.x - Default values for epi v1.2.0
  - For use with epi v1.1.2 or earlier, see comments at `config.apihub` in [values.yaml](values.yaml).
  - For SSO, see comments at `config.apihub`, `config.demiurgeMode` and `config.dsuFabricMode` in [values.yaml](values.yaml).

- From 0.1.x to 0.2.x - Technical release: Significant changes! Please uninstall old versions first! Upgrade from 0.1.x not tested and not guaranteed!
  - Uses Helm hooks for Init and Cleanup
  - Optimized Build process: SeedsBackup will only be created if the underlying Container image has changed, e.g. in case of an upgrade!
  - Readiness probe implemented. Application container is considered as *ready* after build process has been finished.
  - Value `config.ethadapterUrl` has changed from `https://ethadapter.my-company.com:3000` to `http://ethadapter.ethadapter:3000` in order to reflect changes in [ethadapter](https://github.com/pharmaledgerassoc/helmchart-ethadapter/tree/epi-improve-build/charts/ethadapter).
  - Value `persistence.storageClassName` has changed from `gp2` to empty string `""` in order to remove pre-defined setting for AWS and to be cloud-agnostic by default.
  - Configurable sleep time between start of apihub and build process (`config.sleepTime`).
  - Configuration options for PersistentVolumeClaim
  - Configuration has been prepared for running as non-root user (commented out yet, see [values.yaml `podSecurityContext` and `securityContext`](./values.yaml)).
  - Minor optimizations at Kubernetes resources, e.g. set sizeLimit of temporary shared volume, explictly set readOnly flags at volumeMounts.

## Overall

The application consists of two docker images: A Builder image and a Runner Image.

- The *Builder Image* builds the SSApps and prepares the external volume. The builder image does not expose any http ports to the outside and does not service traffic. It will be executed by a Kubernetes Job (aka *Builder Job*) and is run once and on upgrades.
- The *Runner Image* contains the ApiHub and serves http traffic to the clients.
- In addition a docker image containing *kubectl* is required by the *Pre-Builder* and *Cleanup* jobs.

## Helm Lifecycle and Kubernetes Resources Lifetime

This helm chart uses Helm [hooks](https://helm.sh/docs/topics/charts_hooks/) in order to install, upgrade and manage the application and its resources.

```mermaid
sequenceDiagram
  participant PIN as pre-install
  participant PUP as pre-upgrade
  participant I as install
  participant U as uninstall
  participant PUN as post-uninstall
  note right of PUN: Note on resources marked with *:<br/>They are created on a helm hook<br/>and will not be deleted after execution<br/>but will be removed by Cleanup Job
  Note over PIN,PUN: PersistentVolumeClaim for Builder and Runner (B and R)*
  Note over PIN,PUN: ServiceAccount for B and R*
  note right of PUP: Note on upgrades:<br/>Pre-Builder and Builder jobs run<br/>1. if 'builder.forceRun: true'<br/>2. if builder image has changed
  Note over PUP:Pre-Builder Job*
  Note over PUP:Pre-Builder ServiceAccount
  Note over PUP:Pre-Builder Role
  Note over PUP:Pre-Builder RoleBinding
  Note over PIN,PUP:Builder Job*
  Note over PIN,PUP:Builder ConfigMaps Builder
  Note over PIN,PUP:Builder Secret (or SecretProviderClass)
  Note over I,U:Runner Deployment
  Note over I,U:Runner ConfigMaps
  Note over I,U:Runner Service
  Note over I,U:Runner Ingress
  Note over I,U:Runner Secret (or SecretProviderClass)
  Note over I,U:Build-Info ConfigMap
  note right of PUN: Note: Cleanup job deletes <br/>1. Pre-Builder Job<br/>2. Builder Job<br/>3. ServiceAccount for B and R<br/>4. PersistentVolumeClaim for B and R (optional)
  Note over PUN:Cleanup Job
  Note over PUN:Cleanup ServiceAccount
  Note over PUN:Cleanup Role
  Note over PUN:Cleanup RoleBinding
```

## PersistentVolumeClaim

A Persistent Volume is mounted to the Builder and Runner Pod.
Therefore a PersistentVolumeClaim (PVC) is deployed by the helm chart at hook `pre-install` with various configuration options (see [values.yaml](values.yaml) at section `persistence`).
The PVC is being deleted by *Cleanup Job* on deletion of the helm chart if `persistence.deleteOnUninstall: true` (default).

If you want to reuse an existing PVC instead of creating a new one, set `persistent.existingClaim`.

## ServiceAccount

A dedicated ServiceAccount is required by the Builder Job and the Runner Pod(s) if you need to inject Secrets via *CSI Secrets Driver* instead of using Kubernetes Secrets. Further information can be found later in this document.
It is being deployed by the helm chart at hook `pre-install` and is being deleted by *Cleanup Job* on deletion of the helm chart.
In order to create the ServiceAccount, set `serviceAccount.create: true` (default is `false`).

## Pre-Builder Job

The *Pre-Builder job* runs on upgrades (if necessary, see *Builder Job* what "necessary" means) before the *Builder job* is being started.
It a) deletes all remaining *Builder Jobs*, its pods and waits for deletion completed and b) scales Runner deployment to zero and waits for all its pods have been deleted.

Doing so prevents any potential data integrity loss as *Builder Pod* and *Runner Pod(s)* are using the same external volume.

Note: The Job is not being deleted after execution triggered by the helm hook. This allows taking a look at the logs after execution. The Job is deleted by the *Cleanup Job* on deletion of the helm chart.

## Builder Job

The *Builder Job* runs on initial installation and on upgrades if necessary.
The term "necessary" means that it will only be executed if the builder image has changed (= new Software version) or if `builder.forceRun: true`.

The *Builder Job* starts the apihub server (`npm run server`), waits for a short (configurable, see `builder.sleepTime`) period of time and then starts the build process which will prepare data on external volume.

Note: The Job is not being deleted after execution triggered by the helm hook. This allows taking a look at the logs after execution. The Job is deleted by the *Cleanup Job* on deletion of the helm chart.

## Cleanup Job

On deletion/uninstall of the helm chart a Kubernetes Job will be deployed to delete unmanaged helm resources created by helm hooks at `pre-install` and/or `pre-upgrade`.

These resources are:

1. Pre-Builder Job - The *Pre-Builder Job* was created on pre-upgrade and will remain after its execution.
2. Builder Job - The *Builder Job* was created on pre-install/pre-upgrade and will remain after its execution.
3. PersistentVolumeClaim - In case the PersistentVolumeClaim shall not be deleted on deletion of the helm release, set `persistence.deleteOnUninstall` to `false`.
4. ServiceAccount - ServiceAccount is required by *Builder Job* and Runner in case secrets or mounted via CSI Secrets Driver.

## Installation

### Quick install with internal service of type ClusterIP

By default, this helm chart installs the Ethereum Adapter Service at an internal ClusterIP Service listening at port 3000.
This is to prevent exposing the service to the internet by accident!

It is recommended to put non-sensitive configuration values in an configuration file and pass sensitive/secret values via commandline.

1. Create configuration file, e.g. *my-config.yaml*

    ```yaml
    config:
      domain: "epipoc"
      subDomain: "epipoc.companyname"
      vaultDomain: "vault.companyname"
      ethadapterUrl: "http://ethadapter.epi-poc-ethadapter:3000"
      buildSecretKey: "SuperStrongPassword"  # pragma: allowlist secret
      overrides:
        bdnsHosts: |-
          # ... content of the BDNS Hosts file ...

    ```

2. Install via helm to namespace `default`

    ```bash
    helm upgrade my-release-name pharmaledgerassoc/epi --version=0.7.13 \
        --install \
        --values my-config.yaml \
    ```

### Expose Service via Load Balancer

In order to expose the service **directly** by an **own dedicated** Load Balancer, just **add** `service.type` with value `LoadBalancer` to your config file (in order to override the default value which is `ClusterIP`).

**Please note:** At AWS using `service.type` = `LoadBalancer` is not recommended any more, as it creates a Classic Load Balancer. Use [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/) with an ingress instead. A full sample is provided later in the docs. Using an Application Load Balancer (managed by AWS LB Controller) increases security (e.g. by using a Web Application Firewall for your http based traffic) and provides more features like hostname, pathname routing or built-in authentication mechanism via OIDC or AWS Cognito.

Configuration file *my-config.yaml*

```yaml
service:
  type: LoadBalancer

config:
  # ... config section keys and values ...
```

There are more configuration options available like customizing the port and configuring the Load Balancer via annotations (e.g. for configuring SSL Listener).

**Also note:** Annotations are very specific to your environment/cloud provider, see [Kubernetes Service Reference](https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws) for more information. For Azure, take a look [here](https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations).

Sample for AWS (SSL and listening on port 1234 instead 80 which is the default):

```yaml
service:
  type: LoadBalancer
  port: 80
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "80"
    # https://docs.aws.amazon.com/de_de/elasticloadbalancing/latest/classic/elb-security-policy-table.html
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"

# further config
```

### AWS Load Balancer Controller: Expose Service via Ingress

Note: You need the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) installed and configured properly.

1. Enable ingress
2. Add *host*, *path* *`/*`* and *pathType* `ImplementationSpecific`
3. Add annotations for AWS LB Controller
4. A SSL certificate at AWS Certificate Manager (either for the hostname, here `epi.mydomain.com` or wildcard `*.mydomain.com`)

Configuration file *my-config.yaml*

```yaml
ingress:
  enabled: true
  # Let AWS LB Controller handle the ingress (default className is alb)
  # Note: Use className instead of annotation 'kubernetes.io/ingress.class' which is deprecated since 1.18
  # For Kubernetes >= 1.18 it is required to have an existing IngressClass object.
  # See: https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation
  className: alb
  hosts:
    - host: epi.mydomain.com
      # Path must be /* for ALB to match all paths
      paths:
        - path: /*
          pathType: ImplementationSpecific
  # For full list of annotations for AWS LB Controller, see https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/
  annotations:
    # The ARN of the existing SSL Certificate at AWS Certificate Manager
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERTIFICATE_ID
    # The name of the ALB group, can be used to configure a single ALB by multiple ingress objects
    alb.ingress.kubernetes.io/group.name: default
    # Specifies the HTTP path when performing health check on targets.
    alb.ingress.kubernetes.io/healthcheck-path: /
    # Specifies the port used when performing health check on targets.
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    # Specifies the HTTP status code that should be expected when doing health checks against the specified health check path.
    alb.ingress.kubernetes.io/success-codes: "200"
    # Listen on HTTPS protocol at port 443 at the ALB
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    # Use internet facing
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Use most current (as of Dec 2021) encryption ciphers
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    # Use target type IP which is the case if the service type is ClusterIP
    alb.ingress.kubernetes.io/target-type: ip

config:
  # ... config section keys and values ...
```

## CSI Secrets Driver

This enables storing the following secret values in one of the supported Secret stores (e.g. Azure Key Vault or AWS Secrets Manager) instead of using Kubernetes Secrets:

- `apihub.json` - If using SSO, *apihub.json* contains SSO configuration like Client ID and Secret
- `roapihub.json` - Contains Apihub Configuration for leaflet Reader
- `env.json` - Contains the secret passphrase for de/encrypting the generated private keys for the wallets.

More information can be found here:

- [https://secrets-store-csi-driver.sigs.k8s.io/](https://secrets-store-csi-driver.sigs.k8s.io/)
- [https://aws.amazon.com/blogs/security/how-to-use-aws-secrets-configuration-provider-with-kubernetes-secrets-store-csi-driver/](https://aws.amazon.com/blogs/security/how-to-use-aws-secrets-configuration-provider-with-kubernetes-secrets-store-csi-driver/)

Sample for AWS:

1. Prepare `env.json`, `apihub.json` and `roapihub.json` files. See [apihub.json](https://github.com/pharmaledgerassoc/epi-workspace/blob/master/apihub-root/external-volume/config/apihub.json.template), [roapihub.json](https://github.com/pharmaledgerassoc/epi-workspace/blob/master/apihub-root/external-volume/config/ro.apihub.json.template) and [env.json](https://github.com/pharmaledgerassoc/epi-workspace/blob/master/env.json) for templates.
2. Create a new Secret in Secrets Manager with two keys `envJson`, `apihubJson` and `roapihubJson`. Note: If you want to create the secret as PlainText, then you must encode the values to JSON String each!

    Sample:

    ```json
    {
        "envJson": "{  \"PSK_TMP_WORKING_DIR\": \"tmp\", <AND MORE IN JSON STRING FORMAT> }",
        "apihubJson": "{  \"storage\": \"../apihub-root\",  <AND MORE IN JSON STRING FORMAT> }",
        "roapihubJson": "{  \"storage\": \"../apihub-root\",  <AND MORE IN JSON STRING FORMAT> }"

    }
    ```

3. Create IAM role with trust relationship to the Kubernetes Service Account and a policy that allows getting the secret value.

    Trust relationship sample:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "AssumeableByOIDC",
              "Effect": "Allow",
              "Principal": {
                  "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_PROVIDER_ID>"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                  "StringLike": {
                      "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_PROVIDER_ID>:sub": "system:serviceaccount:<K8S_NAMESPACE>:<NAME>"
                  }
              }
          }
      ]
    }
    ```

    Policy:

    ```json
    {
        "Statement": [
            {
                "Action": [
                    "secretsmanager:GetSecretValue",
                    "secretsmanager:DescribeSecret"
                ],
                "Effect": "Allow",
                "Resource": [
                    "<SECRET_ARN>"
                ]
            }
        ],
        "Version": "2012-10-17"
    }
    ```

4. Modify config to use ServiceAccount and SecretProviderClass

    ```yaml
    serviceAccount:
      create: true
      annotations:
        eks.amazonaws.com/role-arn: "<ARN of the IAM role>"

    secretProviderClass:
      enabled: true
      spec:
        provider: aws
        parameters:
          objects: |
            - objectName: "<ARN or Name of Secret>"
              objectType: "secretsmanager"
              jmesPath:
                - path: envJson
                  objectAlias: env.json
                - path: apihubJson
                  objectAlias: apihub.json
                - path: roapihubJson
                  objectAlias: roapihub.json
    ```

## Backup: Create VolumeSnapshot before upgrading and before deletion of helm release

Note: Ensure Volume Snapshotting has been set up appropriately.

```yaml
volumeSnapshots:
  preUpgradeEnabled: true
  finalSnapshotEnabled: true
  # It is stongly recommended to use a class with 'deletionPolicy: Retain' for the pre-upgrade and final snapshots.
  # Otherwise you will loose the snapshot on your storage system if the Kubernetes VolumeSnapshot will be deleted (e.g. if the namespace will be deleted).
  className: "<Name of the VolumeSnapshotClass>"

```

## Additional helm options

Run `helm upgrade --helm` for full list of options.

1. Install to other namespace

    You can install into other namespace than `default` by setting the `--namespace` parameter, e.g.

    ```bash
    helm upgrade my-release-name pharmaledgerassoc/epi --version=0.7.13 \
        --install \
        --namespace=my-namespace \
        --values my-config.yaml \
    ```

2. Wait until installation has finished successfully and the deployment is up and running.

    Provide the `--wait` argument and time to wait (default is 5 minutes) via `--timeout`

    ```bash
    helm upgrade my-release-name pharmaledgerassoc/epi --version=0.7.13 \
        --install \
        --wait --timeout=600s \
        --values my-config.yaml \
    ```

## Potential issues

1. `Error: admission webhook "vingress.elbv2.k8s.aws" denied the request: invalid ingress class: IngressClass.networking.k8s.io "alb" not found`

    **Description:** This error only applies to Kubernetes >= 1.18 and indicates that no matching *IngressClass* object was found.

    **Solution:** Either declare an appropriate IngressClass or omit *className* and add annotation `kubernetes.io/ingress.class`

    Further information:

     - [Kubernetes IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)
     - [AWS Load Balancer controller documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/ingress_class/)

## Helm Unittesting

[helm-unittest](https://github.com/quintush/helm-unittest) is being used for testing the output of the helm chart.
Tests can be found in [tests](./tests)

## Values

*Note:* Please scroll horizontally to show more columns (e.g. description)!

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{"podAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":[{"labelSelector":{"matchExpressions":[{"key":"app.kubernetes.io/name","operator":"In","values":["epi","epiLeafletReader"]}]},"topologyKey":"kubernetes.io/hostname"}]},"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"app.kubernetes.io/name","operator":"In","values":["quorum"]}]},"topologyKey":"kubernetes.io/hostname"},"weight":100}]}}` | Affinity for scheduling a pod. See [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) |
| builder.forceRun | bool | `true` | Boolean flag whether to enforce running the Builder even if it is not required. Useful for testing purpose. |
| builder.image.pullPolicy | string | `"Always"` | Image Pull Policy for the builder. |
| builder.image.repository | string | `"public.ecr.aws/s6w3n1u5/epi-builder"` | The repository of the container image for the builder. <!-- # pragma: allowlist secret --> |
| builder.image.sha | string | `"5ac4de463369b3165b2ec1620b09d07b34b8762fd74add9c17ef3781db507076"` | sha256 digest of the image for the builder. Do not add the prefix "@sha256:" Default to v1.3.1 <!-- # pragma: allowlist secret --> |
| builder.image.tag | string | `"4.0.0"` | Image tag for the builder. Default to v1.3.1 |
| builder.podSecurityContext | object | `{"fsGroup":1000,"runAsGroup":1000,"runAsUser":1000}` | Pod Security Context for the builder. See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) |
| builder.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":false,"runAsGroup":1000,"runAsNonRoot":true,"runAsUser":1000}` | Security Context for the builder container See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container) |
| builder.sleepTime | string | `"10s"` | The time to sleep between start of apihub (npm run server) and build process (npm run build-all) |
| builder.ttlSecondsAfterFinished | int | `600` | Time to keep the Job after finished. If value is not set, then 'ttlSecondsAfterFinished' will not be set. |
| config.buildSecretKey | string | `""` | Secret Pass Phrase for de/encrypting private keys for application wallets created by builder. |
| config.companyName | string | `"Company Inc"` | A CompanyName which is displayed on the web page. |
| config.dev | string | `"false"` | Enable Dev mode |
| config.domain | string | `"epipoc"` | The Domain, e.g. "epipoc" |
| config.enclaveType | string | `"WalletDBEnclave"` | DSU fabric enclave type |
| config.epiVersion | string | `"4.0.0"` | The epi version |
| config.ethadapterUrl | string | `"http://ethadapter.ethadapter:3000"` | The Full URL of the Ethadapter including protocol and port, e.g. "https://ethadapter.my-company.com:3000" |
| config.overrides.apihubJson | string | `""` | Option to explitly set the apihub.json instead of using the default from [https://github.com/pharmaledgerassoc/epi-workspace/blob/v1.3.1/apihub-root/external-volume/config/apihub.json](https://github.com/pharmaledgerassoc/epi-workspace/blob/v1.3.1/apihub-root/external-volume/config/apihub.json). Note: If secretProviderClass.enabled=true, then this value is ignored as it is used/mounted from Secret Vault. <br/> Settings: [https://docs.google.com/document/d/1mg35bb1UBUmTpL1Kt4GuZ7P0K_FMqt2Mb8B3iaDf52I/edit#heading=h.z84gh8sclah3](https://docs.google.com/document/d/1mg35bb1UBUmTpL1Kt4GuZ7P0K_FMqt2Mb8B3iaDf52I/edit#heading=h.z84gh8sclah3) <br/> For SSO (not enabled by default): <br/> 1. "enableOAuth": true <br/> 2. "serverAuthentication": true <br/> 3. For SSO via OAuth with Azure AD, replace <TODO_*> with appropriate values.    For other identity providers (IdP) (e.g. Google, Ping, 0Auth), refer to documentation.    "redirectPath" must match the redirect URL configured at IdP <br/> 4. Add these values to "skipOAuth": "/leaflet-wallet/", "/directory-summary/", "/iframe/" |
| config.overrides.bdnsHosts | string | `""` | Centrally managed and provided BDNS Hosts Config. You must set this value in a non-sandbox environment! See [templates/_configmap-bdns.tpl](templates/_configmap-bdns.tpl) for default value. |
| config.overrides.demiurgeEnvironmentJs | string | `""` | Option to explicitly override the environment.js file used for demiurge-wallet instead of using the predefined template. Note: Usually not required |
| config.overrides.domainConfigJson | string | `""` | Option to explicitly override the config.json used for the domain instead of using the predefined template. Note: Usually not required |
| config.overrides.dsuExplorerEnvironmentJs | string | `""` | Option to explicitly override the environment.js file used for DSU Explorer Wallet instead of using the predefined template. Note: Usually not required |
| config.overrides.dsuFabricEnvironmentJs | string | `""` | Option to explicitly override the environment.js file used for DSU Fabric Wallet instead of using the predefined template. Note: Usually not required |
| config.overrides.envJson | string | `""` | Option to explitly override the env.json for APIHub instead of using the predefined template. Note 1: Usually not required to override. Note 2: If secretProviderClass.enabled=true, then this value is ignored as it is used/mounted from Secret Vault. |
| config.overrides.leafletEnvironmentJs | string | `""` | Option to explicitly override the environment.js file used for Leaflet Wallet instead of using the predefined template. Note: Usually not required |
| config.overrides.lpwaEnvironmentJs | string | `""` | Option to explicitly override the environment.js file used for Lightweight PWA instead of using the predefined template. Note: Usually not required |
| config.overrides.readOnlyApihubJson | string | `""` |  |
| config.overrides.subDomainConfigJson | string | `""` | Option to explicitly override the config.json used for the subDomain instead of using the predefined template. Note: Usually not required |
| config.overrides.vaultDomainConfigJson | string | `""` | Option to explicitly override the config.json used for the vaultDomain instead of using the predefined template. Note: Usually not required |
| config.ssoSecretsEncryptionKey | string | `""` | Base64 encoded 32 bytes string |
| config.subDomain | string | `"epipoc.my-company"` | The Subdomain, should be domain.company, e.g. epipoc.my-company |
| config.vaultDomain | string | `"vault.my-company"` | The Vault domain, should be vault.company, e.g. vault.my-company |
| extraResources | string | `nil` | An array of extra resources that will be deployed. This is useful e.g. for custom resources like SnapshotSchedule provided by [https://github.com/backube/snapscheduler](https://github.com/backube/snapscheduler). |
| fullnameOverride | string | `""` | fullnameOverride completely replaces the generated name. From [https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm](https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm) |
| imagePullSecrets | list | `[]` | Secret(s) for pulling an container image from a private registry. Used for all images. See [https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) |
| ingress.annotations | object | `{}` | Ingress annotations. <br/> For AWS LB Controller, see [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/) <br/> For Azure Application Gateway Ingress Controller, see [https://azure.github.io/application-gateway-kubernetes-ingress/annotations/](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/) <br/> For NGINX Ingress Controller, see [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) <br/> For Traefik Ingress Controller, see [https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations) |
| ingress.className | string | `""` | The className specifies the IngressClass object which is responsible for that class. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) <br/> For Kubernetes >= 1.18 it is required to have an existing IngressClass object. If IngressClass object does not exists, omit className and add the deprecated annotation 'kubernetes.io/ingress.class' instead. <br/> For Kubernetes < 1.18 either use className or annotation 'kubernetes.io/ingress.class'. |
| ingress.enabled | bool | `false` | Whether to create ingress or not for the runner. <br/> Note: For ingress an Ingress Controller (e.g. AWS LB Controller, NGINX Ingress Controller, Traefik, ...) is required and service.type should be ClusterIP or NodePort depending on your configuration |
| ingress.hosts | list | `[{"host":"epi.some-pharma-company.com","paths":[{"path":"/gtinOwner/*","pathType":"ImplementationSpecific"},{"path":"/leaflets/*","pathType":"ImplementationSpecific"},{"path":"/metadata/*","pathType":"ImplementationSpecific"}]}]` | A list of hostnames and path(s) to listen at the Ingress Controller |
| ingress.hosts[0].host | string | `"epi.some-pharma-company.com"` | The FQDN/hostname |
| ingress.hosts[0].paths[0].path | string | `"/gtinOwner/*"` | The Ingress Path. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) <br/> Note: For Ingress Controllers like AWS LB Controller see their specific documentation. |
| ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` | The type of path. This value is required since Kubernetes 1.18. <br/> For Ingress Controllers like AWS LB Controller or Traefik it is usually required to set its value to ImplementationSpecific See [https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/) |
| ingress.hosts[0].paths[1].path | string | `"/leaflets/*"` | The Ingress Path. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) <br/> Note: For Ingress Controllers like AWS LB Controller see their specific documentation. |
| ingress.hosts[0].paths[1].pathType | string | `"ImplementationSpecific"` | The type of path. This value is required since Kubernetes 1.18. <br/> For Ingress Controllers like AWS LB Controller or Traefik it is usually required to set its value to ImplementationSpecific See [https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/) |
| ingress.hosts[0].paths[2].path | string | `"/metadata/*"` | The Ingress Path. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) <br/> Note: For Ingress Controllers like AWS LB Controller see their specific documentation. |
| ingress.hosts[0].paths[2].pathType | string | `"ImplementationSpecific"` | The type of path. This value is required since Kubernetes 1.18. <br/> For Ingress Controllers like AWS LB Controller or Traefik it is usually required to set its value to ImplementationSpecific See [https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/) |
| ingress.tls | list | `[]` |  |
| ingressPrivate.annotations | object | `{}` | Ingress annotations. <br/> For AWS LB Controller, see [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/) <br/> For Azure Application Gateway Ingress Controller, see [https://azure.github.io/application-gateway-kubernetes-ingress/annotations/](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/) <br/> For NGINX Ingress Controller, see [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) <br/> For Traefik Ingress Controller, see [https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/#annotations) |
| ingressPrivate.className | string | `""` | The className specifies the IngressClass object which is responsible for that class. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) <br/> For Kubernetes >= 1.18 it is required to have an existing IngressClass object. If IngressClass object does not exists, omit className and add the deprecated annotation 'kubernetes.io/ingress.class' instead. <br/> For Kubernetes < 1.18 either use className or annotation 'kubernetes.io/ingress.class'. |
| ingressPrivate.enabled | bool | `false` | Whether to create ingress or not for the runner. <br/> Note: For ingress an Ingress Controller (e.g. AWS LB Controller, NGINX Ingress Controller, Traefik, ...) is required and service.type should be ClusterIP or NodePort depending on your configuration |
| ingressPrivate.hosts | list | `[{"host":"epi.some-pharma-company.com","paths":[{"path":"/*","pathType":"ImplementationSpecific"}]}]` | A list of hostnames and path(s) to listen at the Ingress Controller |
| ingressPrivate.hosts[0].host | string | `"epi.some-pharma-company.com"` | The FQDN/hostname |
| ingressPrivate.hosts[0].paths[0].path | string | `"/*"` | The Ingress Path. See [https://kubernetes.io/docs/concepts/services-networking/ingress/#examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) <br/> Note: For Ingress Controllers like AWS LB Controller see their specific documentation. |
| ingressPrivate.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` | The type of path. This value is required since Kubernetes 1.18. <br/> For Ingress Controllers like AWS LB Controller or Traefik it is usually required to set its value to ImplementationSpecific See [https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/) |
| ingressPrivate.tls | list | `[]` |  |
| kubectl.image.pullPolicy | string | `"Always"` | Image Pull Policy |
| kubectl.image.repository | string | `"bitnami/kubectl"` | The repository of the container image containing kubectl |
| kubectl.image.sha | string | `"bba32da4e7d08ce099e40c573a2a5e4bdd8b34377a1453a69bbb6977a04e8825"` | sha256 digest of the image. Do not add the prefix "@sha256:" <br/> Defaults to image digest for "bitnami/kubectl:1.21.14", see [https://hub.docker.com/layers/kubectl/bitnami/kubectl/1.21.14/images/sha256-f9814e1d2f1be7f7f09addd1d877090fe457d5b66ca2dcf9a311cf1e67168590?context=explore](https://hub.docker.com/layers/kubectl/bitnami/kubectl/1.21.14/images/sha256-f9814e1d2f1be7f7f09addd1d877090fe457d5b66ca2dcf9a311cf1e67168590?context=explore) <!-- # pragma: allowlist secret --> |
| kubectl.image.tag | string | `"1.21.14"` | The Tag of the image containing kubectl. Minor Version should match to your Kubernetes Cluster Version. |
| kubectl.podSecurityContext | object | `{"fsGroup":65534,"runAsGroup":65534,"runAsUser":65534}` | Pod Security Context for the pod running kubectl. See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) |
| kubectl.resources | object | `{"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}` | Resource constraints for the pre-builder and cleanup job |
| kubectl.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65534,"runAsNonRoot":true,"runAsUser":65534}` | Security Context for the container running kubectl See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container) |
| kubectl.ttlSecondsAfterFinished | int | `300` | Time to keep the Job after finished in case of an error. If no error occured the Jobs will immediately by deleted. If value is not set, then 'ttlSecondsAfterFinished' will not be set. |
| leafletReader.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].key | string | `"app.kubernetes.io/name"` |  |
| leafletReader.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].operator | string | `"In"` |  |
| leafletReader.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].values[0] | string | `"epi"` |  |
| leafletReader.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].topologyKey | string | `"kubernetes.io/hostname"` |  |
| leafletReader.enabled | bool | `true` | enabling the leaflet-reader installation (apihub in READ MODE, using the data from the same pvc as the normal epi-runner) |
| leafletReader.replicaCount | int | `1` |  |
| nameOverride | string | `""` | nameOverride replaces the name of the chart in the Chart.yaml file, when this is used to construct Kubernetes object names. From [https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm](https://stackoverflow.com/questions/63838705/what-is-the-difference-between-fullnameoverride-and-nameoverride-in-helm) |
| namespaceOverride | string | `""` | Override the deployment namespace. Very useful for multi-namespace deployments in combined charts |
| nodeSelector | object | `{}` | Node Selectors in order to assign pods to certain nodes. See [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) |
| persistence.accessModes | list | `["ReadWriteOnce"]` | AccessModes for the new PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) |
| persistence.dataSource | object | `{}` | DataSource option for cloning an existing volume or creating from a snapshot for a new PVC. See [values.yaml](values.yaml) for more details. |
| persistence.deleteOnUninstall | bool | `true` | Boolean flag whether to delete the (new) PVC on uninstall or not. |
| persistence.existingClaim | string | `""` | The name of an existing PVC to use instead of creating a new one. |
| persistence.finalizers | list | `["kubernetes.io/pvc-protection"]` | Finalizers for the new PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection) |
| persistence.selectorLabels | object | `{}` | Selector Labels for the new PVC. See [https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector) |
| persistence.size | string | `"100Gi"` | Size of the volume for the new PVC |
| persistence.storageClassName | string | `""` | Name of the storage class for the new PVC. If empty or not set then storage class will not be set - which means that the default storage class will be used. |
| resources | object | `{}` | Resource constraints for the builder and runner |
| runner.deploymentStrategy | object | `{"type":"Recreate"}` | The strategy of the deployment for the runner. Defaults to type: Recreate as a PVC is bound to it. See `kubectl explain deployment.spec.strategy` for more and [https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) |
| runner.image.pullPolicy | string | `"Always"` | Image Pull Policy for the runner |
| runner.image.repository | string | `"public.ecr.aws/s6w3n1u5/epi-runner"` | The repository of the container image for the runner <!-- # pragma: allowlist secret --> |
| runner.image.sha | string | `"b81a41ee0a9eb108b167fbcc5027f52ec8e3c3691af77cf1d47daeb2c5cd1530"` | sha256 digest of the image. Do not add the prefix "@sha256:" Default to v1.3.1 <!-- # pragma: allowlist secret --> |
| runner.image.tag | string | `"4.0.0"` | Overrides the image tag whose default is the chart appVersion. Default to v1.3.1 |
| runner.livenessProbe | object | `{"failureThreshold":3,"httpGet":{"httpHeaders":[{"name":"Host","value":"localhost"}],"path":"/installation-details","port":"http"},"initialDelaySeconds":10,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1}` | Liveness probe. See [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| runner.podAnnotations | object | `{}` | Annotations added to the runner pod |
| runner.podSecurityContext | object | `{"fsGroup":1000,"runAsGroup":1000,"runAsUser":1000}` | Pod Security Context for the runner. See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) |
| runner.readinessProbe | object | `{"failureThreshold":3,"httpGet":{"httpHeaders":[{"name":"Host","value":"localhost"}],"path":"/ready-probe","port":"http"},"initialDelaySeconds":10,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1}` | Readiness probe. See [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| runner.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":false,"runAsGroup":1000,"runAsNonRoot":true,"runAsUser":1000}` | Security Context for the runner container See [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container) |
| secretProviderClass.apiVersion | string | `"secrets-store.csi.x-k8s.io/v1"` | API Version of the SecretProviderClass |
| secretProviderClass.enabled | bool | `false` | Whether to use CSI Secrets Store (e.g. Azure Key Vault) instead of "traditional" Kubernetes Secret. NOTE: DO ENABLE, NOT TESTED YET! |
| secretProviderClass.spec | object | `{}` | Spec for the SecretProviderClass. Note: The orgAccountJson must be mounted as objectAlias orgAccountJson |
| service.annotations | object | `{}` | Annotations for the service. See AWS, see [https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws](https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws) For Azure, see [https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations](https://kubernetes-sigs.github.io/cloud-provider-azure/topics/loadbalancer/#loadbalancer-annotations) |
| service.loadBalancerIP | string | `""` | A static IP address for the LoadBalancer if type is LoadBalancer. Note: This only applies to certain Cloud providers like Google or [Azure](https://docs.microsoft.com/en-us/azure/aks/static-ip). [https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer](https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer). |
| service.loadBalancerSourceRanges | string | `nil` | A list of CIDR ranges to whitelist for ingress traffic to the service if type is LoadBalancer. If list is empty, Kubernetes allows traffic from 0.0.0.0/0 |
| service.port | int | `80` | Port where the service will be exposed |
| service.type | string | `"ClusterIP"` | Either ClusterIP, NodePort or LoadBalancer for the runner See [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/) |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.automountServiceAccountToken | bool | `false` | Whether automounting API credentials for a service account is enabled or not. See [https://docs.bridgecrew.io/docs/bc_k8s_35](https://docs.bridgecrew.io/docs/bc_k8s_35) |
| serviceAccount.create | bool | `false` | Specifies whether a service account should be created which is used by builder and runner. |
| serviceAccount.name | string | `""` | The name of the service account to use. If not set and create is true, a name is generated using the fullname template |
| tolerations | list | `[]` | Tolerations for scheduling a pod. See [https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) |
| volumeSnapshots.apiVersion | string | `"v1"` | API Version of the "snapshot.storage.k8s.io" resource. See [https://kubernetes.io/docs/concepts/storage/volume-snapshots/](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) |
| volumeSnapshots.className | string | `""` | The Volume Snapshot class name to use for the pre-upgrade and the final snapshot. It is stongly recommended to use a class with 'deletionPolicy: Retain' for the pre-upgrade and final snapshots. Otherwise you will loose the snapshot on your storage system if the Kubernetes VolumeSnapshot will be deleted (e.g. if the namespace will be deleted). See [https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/) |
| volumeSnapshots.finalSnapshotEnabled | bool | `false` | Whether to create final snapshot before delete. The name of the VolumeSnapshot will be "<helm release name>-final-<UTC timestamp YYYYMMDDHHMM>", e.g. "epi-final-202206221213" |
| volumeSnapshots.preUpgradeEnabled | bool | `false` | Whether to create snapshots before helm upgrading or not. The name of the VolumeSnapshot will be "<helm release name>-upgrade-to-revision-<helm revision>-<UTC timestamp YYYYMMDDHHMM>", e.g. "epi-upgrade-to-revision-19-202206221211" |
| volumeSnapshots.waitForReadyToUse | bool | `true` | Whether to wait until the VolumeSnapshot is ready to use. Note: On first snapshot this may take a while. |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)
