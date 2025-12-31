# SWIM OpenShift Operator

> ⚠️ **Code Under Review** — This repository is part of a reference implementation currently under technical review. Source code will be available soon.

Kubernetes Operator that automates the deployment and lifecycle management of SWIM DNOTAM components on OpenShift. Abstracts infrastructure complexity into simple Custom Resources, enabling aviation stakeholders to deploy compliant SWIM services without deep Kubernetes expertise.

## Overview

The SWIM Operator transforms complex multi-component deployments into single Custom Resource definitions. One CR deploys the complete stack: application, database, message broker, certificates, routes, and observability.

> **Why OpenShift?** This operator is purpose-built for **Red Hat OpenShift**, leveraging enterprise-supported middleware (Red Hat AMQ, Red Hat Build of Keycloak), certified operators (AMQ Broker Operator, Strimzi), and platform-native capabilities (OLM, Routes, RBAC, SecurityContextConstraints). The result is a **tested, validated, and fully supported** deployment path — ideal for aviation stakeholders who require enterprise-grade reliability and vendor accountability. For vanilla Kubernetes deployments without enterprise support, see [swim-kubernetes-operator](https://github.com/swim-developer/swim-kubernetes-operator).

## Custom Resource Definitions

| CRD | Short Name | Description |
|-----|------------|-------------|
| `SwimDigitalNotamProvider` | `sdnp` | Complete DNOTAM Provider stack |
| `SwimDigitalNotamConsumer` | `sdnc` | Complete DNOTAM Consumer stack |
| `SwimDnotamMockServer` | `sdms` | AISP simulation for testing |
| `SwimDnotamMockClient` | `sdmc` | Interactive test client |

## What Gets Deployed

### SwimDigitalNotamProvider

| Component | Description |
|-----------|-------------|
| **Provider App** | Quarkus application with SWIM API |
| **PostgreSQL** | Subscription and event persistence |
| **ActiveMQ Artemis** | AMQP 1.0 broker with OIDC auth |
| **Kafka Topics** | Event distribution (via Strimzi) |
| **TLS Certificates** | Auto-provisioned via cert-manager |
| **Routes** | Edge and Passthrough HTTPS |

### SwimDigitalNotamConsumer

| Component | Description |
|-----------|-------------|
| **Consumer App** | Quarkus application with GraphQL |
| **MongoDB** | Event and subscription storage |
| **Kafka Topics** | Internal event distribution |
| **TLS Certificates** | Client certificates for mTLS |

### SwimDnotamMockServer

| Component | Description |
|-----------|-------------|
| **MockServer App** | AISP simulation with event generator |
| **MariaDB** | Subscription persistence |
| **ActiveMQ Artemis** | AMQP broker for events |
| **TLS Certificates** | Server and client certificates |

### SwimDnotamMockClient

| Component | Description |
|-----------|-------------|
| **MockClient App** | Interactive web UI |
| **MariaDB** | Message persistence |
| **TLS Certificates** | Client certificates for mTLS |

## Standards Alignment

| Standard | Status |
|----------|--------|
| SWIM-TI Yellow Profile | ✅ Implemented, ⏳ Pending Validation |
| Operator Lifecycle Manager (OLM) | ✅ Implemented |
| cert-manager Integration | ✅ Implemented |
| AMQ Broker Operator Integration | ✅ Implemented |
| Strimzi Kafka Integration | ✅ Implemented |

## Prerequisites

- OpenShift 4.12+
- cert-manager Operator
- AMQ Broker Operator
- Strimzi Kafka Operator (for Provider/Consumer)
- SWIM CA ClusterIssuer configured

## Installation

### Via OLM (Recommended)

#### Install the catalog
`oc apply -f install-swim-catalog.yaml`

#### Create subscription
```shell
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: swim-operator
  namespace: openshift-operators
spec:
  channel: alpha
  name: swim-operator
  source: swim-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

### Via Makefile

#### Install CRDs
`make install`

#### Deploy operator
`make deploy IMG=quay.io/masales/swim-operator:latest`

## Quick Start

### Deploy MockServer (AISP Simulation)

```yaml
apiVersion: apps.swim-developer.github.io/v1alpha1
kind: SwimDnotamMockServer
metadata:
  name: mockserver
spec:
  appConfig:
    amqp:
      username: admin
      password: admin
      port: 5672
  certManager:
    issuerName: swim-ca-issuer
    issuerKind: ClusterIssuer
  replicaCount: 1
```  

### Deploy Consumer
```yaml
apiVersion: apps.swim-developer.github.io/v1alpha1
kind: SwimDigitalNotamConsumer
metadata:
  name: swim-dnotam-consumer
spec:
  certManager:
    issuerName: swim-ca-issuer
    issuerKind: ClusterIssuer
  client:
    config:
      swimServiceBaseURL: "https://provider-api.example.com"
      amqpBrokerHost: "broker.example.com"
      amqpBrokerPort: 443
      dnotamSubscriptions: |
        [
          {
            "topic": "DigitalNOTAMService",
            "eventScenario": ["RWY.CLS", "AD.CLS"],
            "airportHeliport": ["LPPT", "EHAM"]
          }
        ]
```

### Deploy Provider
```yaml
apiVersion: apps.swim-developer.github.io/v1alpha1
kind: SwimDigitalNotamProvider
metadata:
  name: swim-dnotam-provider
spec:
  certManager:
    issuerName: swim-ca-issuer
    issuerKind: ClusterIssuer
  artemis:
    adminUser: admin
    adminPassword: admin
    oidc:
      authServerUrl: "https://keycloak.example.com/"
      realm: swim
      clientId: amq-broker
      clientSecret: "<secret>"
  provider:
    oidc:
      authServerUrl: "https://keycloak.example.com/realms/swim"
      clientId: swim-dnotam-provider
      clientSecret: "<secret>"
```      

### Deploy MockClient
```yaml
apiVersion: apps.swim-developer.github.io/v1alpha1
kind: SwimDnotamMockClient
metadata:
  name: dnotam-mockclient
spec:
  keycloak:
    url: "https://keycloak.example.com/"
    realm: swim
    clientId: swim-public-client
  providerAPIURLs: "https://provider-api.example.com"
  amqp:
    host: "broker.example.com"
    port: 443
  mtls:
    enabled: true
    certsSecretName: "swim-dnotam-client-tls"
    passwordsSecretName: "swim-dnotam-mockclient-passwords"
```    

## Operator Features

| Feature | Description |
|---------|-------------|
| **Auto TLS** | Certificates provisioned via cert-manager |
| **Cluster Domain Detection** | Automatic ingress host configuration |
| **Secret Management** | Credentials stored in Kubernetes Secrets |
| **Health Monitoring** | Status conditions on all CRs |
| **Reconciliation** | Continuous state management |
| **OIDC Integration** | Keycloak authentication for Artemis |

## Technology Stack

- **Language**: Go 1.24+
- **Framework**: Kubebuilder / Operator SDK
- **API Version**: `apps.swim-developer.github.io/v1alpha1`
- **Distribution**: OLM Bundle

## Development

# Run locally (outside cluster)
`make run`

# Run tests
`make test`

# Build container image
`make docker-build IMG=quay.io/yourorg/swim-operator:tag`

# Push image
`make docker-push IMG=quay.io/yourorg/swim-operator:tag`

# Generate bundle for OLM
`make bundle`

## Uninstall

### Delete all CRs first
```shell
oc delete swimdnotammockserver --all
oc delete swimdigitalnotamprovider --all
oc delete swimdigitalnotamconsumer --all
oc delete swimdnotammockclient --all
```

### Undeploy operator
`make undeploy`

### Remove CRDs
`make uninstall`

## License

BSD 3-Clause License
