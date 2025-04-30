# CouchDB

![Version: 4.6.1](https://img.shields.io/badge/Version-4.6.1-informational?style=flat-square) ![AppVersion: 3.3.3](https://img.shields.io/badge/AppVersion-3.3.3-informational?style=flat-square)

Apache CouchDB is a database featuring seamless multi-master sync, that scales
from big data to mobile, with an intuitive HTTP/JSON API and designed for
reliability.

This chart deploys a CouchDB cluster as a StatefulSet. It creates a ClusterIP
Service in front of the Deployment for load balancing by default, but can also
be configured to deploy other Service types or an Ingress Controller. The
default persistence mechanism is simply the ephemeral local filesystem, but
production deployments should set `persistentVolume.enabled` to `true` to attach
storage volumes to each Pod in the Deployment.

## TL;DR

```bash
$ helm repo add pharmaledgerassoc https://pharmaledgerassoc.github.io/helm-charts
$ helm install pharmaledgerassoc/couchdb \
  --version=4.6.1 \
  --set allowAdminParty=true \
  --set couchdbConfig.couchdb.uuid=$(curl https://www.uuidgenerator.net/api/version4 2>/dev/null | tr -d -)
```

## Prerequisites

- Kubernetes 1.9+ with Beta APIs enabled
- Ingress requires Kubernetes 1.19+

## Installing the Chart

To install the chart with the release name `my-release`:

Add the CouchDB Helm repository:

```bash
$ helm repo add pharmaledger https://pharmaledgerassoc.github.io/helm-charts
```

Afterwards install the chart replacing the UUID
`decafbaddecafbaddecafbaddecafbad` with a custom one:

```bash
$ helm install \
  --name my-release \
  --version=4.6.1 \
  --set couchdbConfig.couchdb.uuid=decafbaddecafbaddecafbaddecafbad \
  pharmaledgerassoc/couchdb
```

This will create a Secret containing the admin credentials for the cluster.
Those credentials can be retrieved as follows:

```bash
$ kubectl get secret my-release-couchdb -o go-template='{{ .data.adminPassword }}' | base64 --decode
```

If you prefer to configure the admin credentials directly you can create a
Secret containing `adminUsername`, `adminPassword` and `cookieAuthSecret` keys:

```bash
$  kubectl create secret generic my-release-couchdb --from-literal=adminUsername=foo --from-literal=adminPassword=bar --from-literal=cookieAuthSecret=baz
```

If you want to set the `adminHash` directly to achieve consistent salts between
different nodes you need to add it to the secret:

```bash
$  kubectl create secret generic my-release-couchdb \
   --from-literal=adminUsername=foo \
   --from-literal=cookieAuthSecret=baz \
   --from-literal=adminHash=-pbkdf2-d4b887da....
```

and then install the chart while overriding the `createAdminSecret` setting:

```bash
$ helm install \
  --name my-release \
  --version=4.6.1 \
  --set createAdminSecret=false \
  --set couchdbConfig.couchdb.uuid=decafbaddecafbaddecafbaddecafbad \
  pharmaledgerassoc/couchdb
```

If you want to use an Secret Provider Class
Must fill the spec on values like this:

```bash
# Settings for using SecretProviderClass (CSI Secrets driver) instead of Secret.
secretProviderClass:
  enabled: true
  spec:
    provider: aws
    parameters:
      objects: |
        - objectName: ${aws_secretsmanager_secret.couchdb.arn}
          objectType: "secretsmanager"
          jmesPath:
            - path: COUCHDB_USER
              objectAlias: COUCHDB_USER
            - path: COUCHDB_PASSWORD
              objectAlias: COUCHDB_PASSWORD
            - path: COUCHDB_READER_PASSWORD
              objectAlias: COUCHDB_READER_PASSWORD
            - path: UUID
              objectAlias: UUID
            - path: COUCHDB_SECRET
              objectAlias: COUCHDB_SECRET
            - path: COUCHDB_ERLANG_COOKIE
              objectAlias: COUCHDB_ERLANG_COOKIE
    secretObjects:
      - secretName: couchdb-couchdb-secrets
        type: Opaque
        data:
          - objectName: COUCHDB_USER
            key: COUCHDB_USER
          - objectName: COUCHDB_PASSWORD
            key: COUCHDB_PASSWORD
          - objectName: COUCHDB_READER_PASSWORD
            key: COUCHDB_READER_PASSWORD
          - objectName: UUID
            key: UUID
          - objectName: COUCHDB_SECRET
            key: COUCHDB_SECRET
          - objectName: COUCHDB_ERLANG_COOKIE
            key: COUCHDB_ERLANG_COOKIE
```

When using SecretProviderClass the secret needs to be created manually on provider.
Example of the secrets required.

```bash
{
  "COUCHDB_ADMIN": "admin",
  "COUCHDB_ADMIN_PASSWORD": "@ocm06vQvnzf?1H6",
  "COUCHDB_READER_PASSWORD": "BfZ*5ToAmJtab*lY",
  "UUID": "5fcb2906-9db6-f587-c1ce-8ca857dcc57c"
}
```

Also add the required anotation on service account to allow retrieve secrets.

```bash
serviceAccount:
  create: true
  name: couchdb
  annotations:
    eks.amazonaws.com/role-arn: ${aws_iam_role.couchdb.arn}
```

To create a read only user after deploy it's required to set it true on values.

```bash
readerUserSetup:
  enabled: true
```

This Helm chart deploys CouchDB on the Kubernetes cluster in a default
configuration. The [configuration](#configuration) section lists
the parameters that can be configured during installation.

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `my-release` Deployment:

```bash
$ helm delete my-release
```

The command removes all the Kubernetes components associated with the chart and
deletes the release.

## Configuration

The following table lists the most commonly configured parameters of the
CouchDB chart and their default values:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| adminPasswordKey | string | `""` |  |
| adminUsername | string | `"admin"` |  |
| adminUsernameKey | string | `""` |  |
| affinity | object | `{}` |  |
| allowAdminParty | bool | `false` | If allowAdminParty is enabled the cluster will start up without any database administrator account; i.e., all users will be granted administrative access. Otherwise, the system will look for a Secret called <ReleaseName>-couchdb containing `adminUsername`, `adminPassword` and `cookieAuthSecret` keys. See the `createAdminSecret` flag. ref: https://kubernetes.io/docs/concepts/configuration/secret/ |
| annotations | object | `{}` |  |
| autoSetup.defaultDatabases[0] | string | `"_global_changes"` |  |
| autoSetup.enabled | bool | `false` |  |
| autoSetup.image.pullPolicy | string | `"Always"` |  |
| autoSetup.image.repository | string | `"curlimages/curl"` |  |
| autoSetup.image.tag | string | `"latest"` |  |
| clusterSize | int | `3` | the initial number of nodes in the CouchDB cluster. |
| couchdbConfig | object | `{"chttpd":{"bind_address":"any","require_valid_user":false}}` | couchdbConfig will override default CouchDB configuration settings. The contents of this map are reformatted into a .ini file laid down by a ConfigMap object. ref: http://docs.couchdb.org/en/latest/config/index.html |
| createAdminSecret | bool | `false` | If createAdminSecret is enabled a Secret called <ReleaseName>-couchdb will be created containing auto-generated credentials. Users who prefer to set these values themselves have a couple of options:  1) The `adminUsername`, `adminPassword`, `adminHash`, and `cookieAuthSecret`    can be defined directly in the chart's values. Note that all of a chart's    values are currently stored in plaintext in a ConfigMap in the tiller    namespace.  2) This flag can be disabled and a Secret with the required keys can be    created ahead of time. |
| dns.clusterDomainSuffix | string | `"cluster.local"` |  |
| enableSearch | bool | `false` | Flip this to flag to include the Search container in each Pod |
| erlangFlags | object | `{"name":"couchdb"}` | erlangFlags is a map that is passed to the Erlang VM as flags using the ERL_FLAGS env. The `name` flag is required to establish connectivity between cluster nodes. ref: http://erlang.org/doc/man/erl.html#init_flags |
| extraEnv | list | `[]` |  |
| extraEnvTemplate | bool | `false` |  |
| extraPorts | list | `[]` |  |
| extraSecretName | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.repository | string | `"couchdb"` |  |
| image.tag | string | `"3.3.3"` |  |
| ingress.annotations | object | `{}` |  |
| ingress.enabled | bool | `false` |  |
| ingress.hosts[0] | string | `"chart-example.local"` |  |
| ingress.path | string | `"/"` |  |
| ingress.tls | string | `nil` |  |
| initImage.pullPolicy | string | `"Always"` |  |
| initImage.repository | string | `"busybox"` |  |
| initImage.tag | string | `"latest"` |  |
| initResources | object | `{}` |  |
| labels | object | `{}` |  |
| lifecycle | object | `{}` |  |
| lifecycleTemplate | bool | `false` |  |
| livenessProbe.enabled | bool | `true` |  |
| livenessProbe.failureThreshold | int | `3` |  |
| livenessProbe.initialDelaySeconds | int | `0` |  |
| livenessProbe.periodSeconds | int | `10` |  |
| livenessProbe.successThreshold | int | `1` |  |
| livenessProbe.timeoutSeconds | int | `1` |  |
| networkPolicy.enabled | bool | `true` |  |
| persistentVolume | object | `{"accessModes":["ReadWriteOnce"],"annotations":{},"enabled":false,"existingClaims":[],"size":"10Gi"}` | The storage volume used by each Pod in the StatefulSet. If a persistentVolume is not enabled, the Pods will use `emptyDir` ephemeral local storage. Setting the storageClass attribute to "-" disables dynamic provisioning of Persistent Volumes; leaving it unset will invoke the default provisioner. |
| persistentVolumeClaimRetentionPolicy.enabled | bool | `false` | Sets the persistentVolumeClaimRetentionPolicy enable for Statefulset PVC |
| persistentVolumeClaimRetentionPolicy.whenDeleted | string | `"Retain"` | Can be set as Retain or Delete, if on Delete the pv will be deleted if the statefulset was removed. |
| persistentVolumeClaimRetentionPolicy.whenScaled | string | `"Retain"` | Must be set as Retain to allow scale down and scale up withour remove the pv |
| placementConfig.enabled | bool | `false` |  |
| placementConfig.image.repository | string | `"caligrafix/couchdb-autoscaler-placement-manager"` |  |
| placementConfig.image.tag | string | `"0.1.0"` |  |
| podDisruptionBudget.enabled | bool | `false` |  |
| podDisruptionBudget.maxUnavailable | int | `1` |  |
| podManagementPolicy | string | `"Parallel"` |  |
| priorityClassName | string | `""` |  |
| prometheusPort.bind_address | string | `"0.0.0.0"` |  |
| prometheusPort.enabled | bool | `false` |  |
| prometheusPort.port | int | `17986` |  |
| readerUserSetup.enabled | bool | `true` |  |
| readinessProbe.enabled | bool | `true` |  |
| readinessProbe.failureThreshold | int | `3` |  |
| readinessProbe.initialDelaySeconds | int | `0` |  |
| readinessProbe.periodSeconds | int | `10` |  |
| readinessProbe.successThreshold | int | `1` |  |
| readinessProbe.timeoutSeconds | int | `1` |  |
| resources | object | `{}` |  |
| searchImage.pullPolicy | string | `"IfNotPresent"` |  |
| searchImage.repository | string | `"kocolosk/couchdb-search"` |  |
| searchImage.tag | string | `"0.2.0"` |  |
| secretProviderClass.apiVersion | string | `"secrets-store.csi.x-k8s.io/v1"` | API Version of the SecretProviderClass |
| secretProviderClass.enabled | bool | `false` |  |
| secretProviderClass.spec | object | `{}` | Spec for the SecretProviderClass. |
| service.annotations | object | `{}` |  |
| service.enabled | bool | `true` |  |
| service.externalPort | int | `5984` |  |
| service.extraPorts | list | `[]` |  |
| service.labels | object | `{}` |  |
| service.targetPort | int | `5984` |  |
| service.type | string | `"ClusterIP"` |  |
| serviceAccount.annotations | object | `{}` | Add required anotations to Service account |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.enabled | bool | `true` |  |
| sidecars | object | `{}` |  |
| tolerations | list | `[]` |  |
| topologySpreadConstraints | object | `{}` |  |

You can set the values of the `couchdbConfig` map according to the
[official configuration][4]. The following shows the map's default values and
required options to set:

|           Parameter             |             Description                                            |                Default                 |
|---------------------------------|--------------------------------------------------------------------|----------------------------------------|
| `couchdb.uuid`                  | UUID for this CouchDB server instance ([Required in a cluster][5]) |                                        |
| `chttpd.bind_address`           | listens on all interfaces when set to any                          | any                                    |
| `chttpd.require_valid_user`     | disables all the anonymous requests to the port 5984 when true     | false                                  |

A variety of other parameters are also configurable. See the comments in the
`values.yaml` file for further details:

| Parameter                            | Default                                                                                                                                                      |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `adminUsername`                      | admin                                                                                                                                                        |
| `adminPassword`                      | auto-generated                                                                                                                                               |
| `adminHash`                          |                                                                                                                                                              |
| `cookieAuthSecret`                   | auto-generated                                                                                                                                               |
| `image.repository`                   | couchdb                                                                                                                                                      |
| `image.tag`                          | 3.3.3                                                                                                                                                        |
| `image.pullPolicy`                   | IfNotPresent                                                                                                                                                 |
| `searchImage.repository`             | kocolosk/couchdb-search                                                                                                                                      |
| `searchImage.tag`                    | 0.1.0                                                                                                                                                        |
| `searchImage.pullPolicy`             | IfNotPresent                                                                                                                                                 |
| `initImage.repository`               | busybox                                                                                                                                                      |
| `initImage.tag`                      | latest                                                                                                                                                       |
| `initImage.pullPolicy`               | Always                                                                                                                                                       |
| `ingress.enabled`                    | false                                                                                                                                                        |
| `ingress.hosts`                      | chart-example.local                                                                                                                                          |
| `ingress.annotations`                |                                                                                                                                                              |
| `ingress.path`                       | /                                                                                                                                                            |
| `ingress.tls`                        |                                                                                                                                                              |
| `persistentVolume.accessModes`       | ReadWriteOnce                                                                                                                                                |
| `persistentVolume.storageClass`      | Default for the Kube cluster                                                                                                                                 |
| `persistentVolume.annotations`       | {}                                                                                                                                                           |
| `podDisruptionBudget.enabled`        | false                                                                                                                                                        |
| `podDisruptionBudget.minAvailable`   | nil                                                                                                                                                          |
| `podDisruptionBudget.maxUnavailable` | 1                                                                                                                                                            |
| `podManagementPolicy`                | Parallel                                                                                                                                                     |
| `affinity`                           |                                                                                                                                                              |
| `topologySpreadConstraints`          |                                                                                                                                                              |
| `labels`                             |                                                                                                                                                              |
| `annotations`                        |                                                                                                                                                              |
| `tolerations`                        |                                                                                                                                                              |
| `resources`                          |                                                                                                                                                              |
| `autoSetup.enabled`                  | false (if set to true, must have `service.enabled` set to true and a correct `adminPassword` - deploy it with the `--wait` flag to avoid first jobs failure) |
| `autoSetup.image.repository`         | alpine/curl                                                                                                                                                  |
| `autoSetup.image.tag`                | latest                                                                                                                                                       |
| `autoSetup.image.pullPolicy`         | Always                                                                                                                                                       |
| `autoSetup.defaultDatabases`         | [`_global_changes`]                                                                                                                                          |
| `service.annotations`                |                                                                                                                                                              |
| `service.enabled`                    | true                                                                                                                                                         |
| `service.type`                       | ClusterIP                                                                                                                                                    |
| `service.externalPort`               | 5984                                                                                                                                                         |
| `service.targetPort`                 | 5984                                                                                                                                                         |
| `dns.clusterDomainSuffix`            | cluster.local                                                                                                                                                |
| `networkPolicy.enabled`              | true                                                                                                                                                         |
| `serviceAccount.enabled`             | true                                                                                                                                                         |
| `serviceAccount.create`              | true                                                                                                                                                         |
| `serviceAccount.imagePullSecrets`    |                                                                                                                                                              |
| `sidecars`                           | {}                                                                                                                                                           |
| `livenessProbe.enabled`              | true                                                                                                                                                         |
| `livenessProbe.failureThreshold`     | 3                                                                                                                                                            |
| `livenessProbe.initialDelaySeconds`  | 0                                                                                                                                                            |
| `livenessProbe.periodSeconds`        | 10                                                                                                                                                           |
| `livenessProbe.successThreshold`     | 1                                                                                                                                                            |
| `livenessProbe.timeoutSeconds`       | 1                                                                                                                                                            |
| `readinessProbe.enabled`             | true                                                                                                                                                         |
| `readinessProbe.failureThreshold`    | 3                                                                                                                                                            |
| `readinessProbe.initialDelaySeconds` | 0                                                                                                                                                            |
| `readinessProbe.periodSeconds`       | 10                                                                                                                                                           |
| `readinessProbe.successThreshold`    | 1                                                                                                                                                            |
| `readinessProbe.timeoutSeconds`      | 1                                                                                                                                                            |
| `prometheusPort.enabled`             | false                                                                                                                                                        |
| `prometheusPort.port`                | 17896                                                                                                                                                        |
| `prometheusPort.bind_address`        | 0.0.0.0                                                                                                                                                      |
| `placementConfig.enabled`            | false                                                                                                                                                        |
| `placementConfig.image.repository`   | caligrafix/couchdb-autoscaler-placement-manager                                                                                                              |
| `placementConfig.image.tag`          | 0.1.0                                                                                                                                                        |
| `podSecurityContext`                 |                                                                                                                                                              |
| `containerSecurityContext            |                                                                                                                                                              |

## Feedback, Issues, Contributing

General feedback is welcome at our [user][1] or [developer][2] mailing lists.

Apache CouchDB has a [CONTRIBUTING][3] file with details on how to get started
with issue reporting or contributing to the upkeep of this project. In short,
use GitHub Issues, do not report anything on Docker's website.

## Non-Apache CouchDB Development Team Contributors

- [@natarajaya](https://github.com/natarajaya)
- [@satchpx](https://github.com/satchpx)
- [@spanato](https://github.com/spanato)
- [@jpds](https://github.com/jpds)
- [@sebastien-prudhomme](https://github.com/sebastien-prudhomme)
- [@stepanstipl](https://github.com/sebastien-stepanstipl)
- [@amatas](https://github.com/amatas)
- [@Chimney42](https://github.com/Chimney42)
- [@mattjmcnaughton](https://github.com/mattjmcnaughton)
- [@mainephd](https://github.com/mainephd)
- [@AdamDang](https://github.com/AdamDang)
- [@mrtyler](https://github.com/mrtyler)
- [@kevinwlau](https://github.com/kevinwlau)
- [@jeyenzo](https://github.com/jeyenzo)
- [@Pinpin31.](https://github.com/Pinpin31)

[1]: http://mail-archives.apache.org/mod_mbox/couchdb-user/
[2]: http://mail-archives.apache.org/mod_mbox/couchdb-dev/
[3]: https://github.com/apache/couchdb/blob/master/CONTRIBUTING.md
[4]: https://docs.couchdb.org/en/stable/config/index.html
[5]: https://docs.couchdb.org/en/latest/setup/cluster.html#preparing-couchdb-nodes-to-be-joined-into-a-cluster
