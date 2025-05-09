# echo-server

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.16.0](https://img.shields.io/badge/AppVersion-1.16.0-informational?style=flat-square)

A Helm chart for Kubernetes

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].key | string | `"topology.kubernetes.io/zone"` |  |
| affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].operator | string | `"In"` |  |
| affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].values[0] | string | `"eu-west-1a"` |  |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.repository | string | `"ealen/echo-server"` |  |
| image.tag | string | `"0.5.1"` |  |
| imagePullSecrets | list | `[]` |  |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| podAnnotations | object | `{}` |  |
| podSecurityContext | object | `{}` |  |
| replicaCount | int | `1` |  |
| resources | object | `{}` |  |
| securityContext | object | `{}` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled" | string | `"false"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-eip-allocations" | string | `"eipalloc-05bbd88e5c4fb7078"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold" | string | `"2"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold" | string | `"2"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-ip-address-type" | string | `"ipv4"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-name" | string | `"echo-server"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-nlb-target-type" | string | `"instance"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-scheme" | string | `"internet-facing"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-subnets" | string | `"eks-ireland-1-vpc-public-eu-west-1a"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-target-group-attributes" | string | `"preserve_client_ip.enabled=true,deregistration_delay.timeout_seconds=120,deregistration_delay.connection_termination.enabled=true,stickiness.enabled=true,stickiness.type=source_ip"` |  |
| service.annotations."service.beta.kubernetes.io/aws-load-balancer-type" | string | `"external"` |  |
| service.loadBalancerSourceRanges[0] | string | `"8.8.8.8/32"` |  |
| service.loadBalancerSourceRanges[1] | string | `"8.8.4.4/32"` |  |
| service.port | int | `80` |  |
| serviceAccount.annotations | object | `{}` |  |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.name | string | `""` |  |
| tolerations | list | `[]` |  |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)
