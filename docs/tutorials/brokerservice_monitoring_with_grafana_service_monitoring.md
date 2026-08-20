---
title: "BrokerService Monitoring with Prometheus and Grafana"
description: "Build and observe a realistic order-processing pipeline using BrokerService, BrokerApp, Camel, Prometheus and Grafana."
draft: false
images: []
menu:
  docs:
    parent: "tutorials"
weight: 123
toc: true
---

## 1. What We're Building

This tutorial deploys a realistic event-driven order-processing pipeline and shows how to observe it with Prometheus and Grafana.

### Architecture

```
 Traffic Generator           5 msg/s
 (order-generator)               │
                                 ▼
                          ORDERS.NEW
                                 │
                          first-app
                          Order Processor
                                 │
                                 ▼
                       ORDERS.PROCESSED
                                 │
                          second-app
                          Shipping Service
                                 │
                                 ▼
                        ORDERS.SHIPPED
                                 │
                          third-app
                          Delivery Service
                                 │
                                 ▼
                       ORDERS.DELIVERED
                                 │
                          master-sink
                          (optional drain)
                                 │
                                 ▼

          BrokerService → Prometheus → Grafana
```

**One reusable Camel image, four pipeline roles.**
The same container image (`camel-jms-app`) is deployed four times for the core pipeline. An optional fifth deployment, `master-sink`, can drain the terminal queue when needed.
Role and queue configuration come from environment variables.

| Kubernetes Deployment | BrokerApp identity | `APP_ROLE` | Consumes | Produces |
|---|---|---|---|---|
| order-generator | `order-generator` | `generator` | — | `ORDERS.NEW` |
| camel-jms-app | `first-app` | `processor` | `ORDERS.NEW` | `ORDERS.PROCESSED` |
| camel-jms-app-second | `second-app` | `shipping` | `ORDERS.PROCESSED` | `ORDERS.SHIPPED` |
| camel-jms-app-third | `third-app` | `delivery` | `ORDERS.SHIPPED` | `ORDERS.DELIVERED` |

The BrokerApp identity (column 2) is what Artemis uses for access control. The Kubernetes Deployment name is what `kubectl` uses. These are intentionally different so the tutorial's operational commands are unambiguous.

### Prerequisites

- A running Kubernetes cluster (this tutorial uses `minikube`)
- `kubectl` configured to interact with your cluster
- `helm` installed for deploying monitoring components

---

## 2. Setup Infrastructure

### Start Minikube

```bash {"stage":"init", "id":"minikube_start", "runtime":"bash"}
minikube start --profile brokerservice-monitoring --cpus 8 --memory 8192 --disk-size 8000
minikube profile brokerservice-monitoring
```

### Create Namespace

```bash {"stage":"init", "runtime":"bash"}
kubectl create namespace service-app-project
kubectl config set-context --current --namespace=service-app-project
```

### Install Cert-Manager

```bash {"stage":"init", "label":"install cert-manager", "runtime":"bash"}
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.1/cert-manager.yaml
```

Wait for `cert-manager` to be ready:

```bash {"stage":"init", "label":"wait for cert-manager", "runtime":"bash"}
kubectl wait deployment --for=condition=Available -n cert-manager --timeout=600s cert-manager cert-manager-cainjector cert-manager-webhook
```

### Install Trust Manager

```bash {"stage":"init", "label":"add jetstack helm repo", "runtime":"bash"}
helm repo add jetstack https://charts.jetstack.io --force-update
```

```bash {"stage":"init", "label":"install trust-manager", "runtime":"bash"}
helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager --set secretTargets.enabled=true --set secretTargets.authorizedSecretsAll=true --wait
```

### Install kube-prometheus-stack

```bash {"stage":"init", "label":"add prometheus helm repo", "runtime":"bash"}
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```bash {"stage":"init", "label":"install kube-prometheus-stack", "runtime":"bash"}
helm upgrade -i prometheus prometheus-community/kube-prometheus-stack \
  -n service-app-project \
  --set grafana.sidecar.dashboards.enabled=true \
  --set grafana.sidecar.dashboards.label=grafana_dashboard \
  --set grafana.sidecar.dashboards.searchNamespace=ALL \
  --set grafana.sidecar.datasources.enabled=true \
  --set kubeEtcd.enabled=false \
  --set kubeControllerManager.enabled=false \
  --set kubeScheduler.enabled=false \
  --wait
```

Wait for all monitoring components:

```bash {"stage":"init", "label":"wait for prometheus stack", "runtime":"bash"}
kubectl wait deployment --for=condition=Available -n service-app-project prometheus-grafana prometheus-kube-prometheus-operator --timeout=300s
kubectl wait statefulset --for=jsonpath='{.status.readyReplicas}'=1 -n service-app-project prometheus-prometheus-kube-prometheus-prometheus --timeout=300s
```

### Install the Operator

```bash {"stage":"init", "rootdir":"$initial_dir", "runtime":"bash"}
./deploy/install_opr.sh
```

```bash {"stage":"init", "label":"wait for the operator to be running", "runtime":"bash"}
kubectl wait deployment arkmq-org-broker-controller-manager --for=create --timeout=240s
kubectl wait pod --all --for=condition=Ready --namespace=service-app-project --timeout=600s
```

---

## 3. Configure Certificates

### Create Issuers and Root Certificate

```bash {"stage":"deploy_certs", "label":"create root issuer", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: root-issuer
spec:
  selfSigned: {}
EOF
```

```bash {"stage":"deploy_certs", "label":"wait for root issuer", "runtime":"bash"}
kubectl wait clusterissuer root-issuer --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_certs", "label":"create root cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: root-cert
  namespace: cert-manager
spec:
  isCA: true
  commonName: artemis.root.ca
  secretName: artemis-root-cert-secret
  issuerRef:
    name: root-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_certs", "label":"wait for root cert", "runtime":"bash"}
kubectl wait certificate root-cert --for=condition=Ready -n cert-manager --timeout=300s
```

```bash {"stage":"deploy_certs", "label":"create signing issuer", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: broker-ca-issuer
spec:
  ca:
    secretName: artemis-root-cert-secret
EOF
```

```bash {"stage":"deploy_certs", "label":"wait for signing issuer", "runtime":"bash"}
kubectl wait clusterissuer broker-ca-issuer --for=condition=Ready --timeout=300s
```

### Create Operator Certificate

```bash {"stage":"deploy_certs", "label":"create ca bundle", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: arkmq-org-broker-manager-ca
  namespace: cert-manager
spec:
  sources:
  - secret:
      name: artemis-root-cert-secret
      key: "tls.crt"
  target:
    secret:
      key: "ca.pem"
EOF
```

```bash {"stage":"deploy_certs", "label":"wait for ca bundle", "runtime":"bash"}
kubectl wait bundle arkmq-org-broker-manager-ca -n cert-manager --for=condition=Synced --timeout=300s
```

```bash {"stage":"deploy_certs", "label":"create operator cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: arkmq-org-broker-manager-cert
  namespace: service-app-project
spec:
  secretName: arkmq-org-broker-manager-cert
  commonName: arkmq-org-broker-operator
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_certs", "label":"wait for operator cert", "runtime":"bash"}
kubectl wait certificate arkmq-org-broker-manager-cert -n service-app-project --for=condition=Ready --timeout=300s
```

---

## 4. Deploy BrokerService and BrokerApps

### BrokerService Certificate

```bash {"stage":"deploy_service", "label":"create broker cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: messaging-service-broker-cert
  namespace: service-app-project
spec:
  secretName: messaging-service-broker-cert
  commonName: messaging-service
  dnsNames:
  - messaging-service
  - messaging-service.service-app-project.svc.cluster.local
  - '*.messaging-service-hdls-svc.service-app-project.svc.cluster.local'
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_service", "label":"wait for broker cert", "runtime":"bash"}
kubectl wait certificate messaging-service-broker-cert -n service-app-project --for=condition=Ready --timeout=300s
```

### Deploy BrokerService

```bash {"stage":"deploy_service", "label":"deploy brokerservice", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerService
metadata:
  name: messaging-service
  namespace: service-app-project
  labels:
    forWorkQueue: "true"
spec:
  resources:
    requests:
      memory: "2Gi"
    limits:
      memory: "2Gi"
  env:
    - name: JAVA_ARGS_APPEND
      value: "-Dlog4j2.level=INFO"
EOF
```

```bash {"stage":"deploy_service", "label":"wait for brokerservice", "runtime":"bash"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=Ready --timeout=300s
```

### Deploy BrokerApps

Each `BrokerApp` declares exactly the permissions its pipeline stage needs. The ownership chain is:

```
order-generator  →  ORDERS.NEW  →  first-app  →  ORDERS.PROCESSED  →  second-app  →  ORDERS.SHIPPED  →  third-app  →  ORDERS.DELIVERED
   (produce)          (consume/produce)                (consume/produce)                  (consume/produce)
```

Each app only owns the addresses it **produces**. Downstream consumers reference upstream producers using `appName` + `appNamespace`.

#### order-generator (Traffic Generator)

The generator has a single capability: produce into `ORDERS.NEW`.

```bash {"stage":"deploy_app", "label":"create order-generator cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: order-generator-app-cert
  namespace: service-app-project
spec:
  secretName: order-generator-app-cert
  commonName: order-generator
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_app", "label":"wait for order-generator cert", "runtime":"bash"}
kubectl wait certificate order-generator-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy order-generator brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: order-generator
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      forWorkQueue: "true"
  sharedAddresses:
    - address: "ORDERS.NEW"
  capabilities:
    - producerOf:
        - address: "ORDERS.NEW"
EOF
```

```bash {"stage":"deploy_app", "label":"wait for order-generator brokerapp", "runtime":"bash"}
kubectl wait BrokerApp order-generator -n service-app-project --for=condition=Ready --timeout=300s
```

#### first-app (Order Processor)

```bash {"stage":"deploy_app", "label":"create first-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: first-app-app-cert
  namespace: service-app-project
spec:
  secretName: first-app-app-cert
  commonName: first-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_app", "label":"wait for first-app cert", "runtime":"bash"}
kubectl wait certificate first-app-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy first-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: first-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      forWorkQueue: "true"
  sharedAddresses:
    - address: "ORDERS.PROCESSED"
  capabilities:
    - consumerOf:
        - address: "ORDERS.NEW"
          appName: "order-generator"
          appNamespace: "service-app-project"
      producerOf:
        - address: "ORDERS.PROCESSED"
EOF
```

```bash {"stage":"deploy_app", "label":"wait for first-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp first-app -n service-app-project --for=condition=Ready --timeout=300s
```

#### second-app (Shipping Service)

```bash {"stage":"deploy_app", "label":"create second-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: second-app-app-cert
  namespace: service-app-project
spec:
  secretName: second-app-app-cert
  commonName: second-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_app", "label":"wait for second-app cert", "runtime":"bash"}
kubectl wait certificate second-app-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy second-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: second-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      forWorkQueue: "true"
  sharedAddresses:
    - address: "ORDERS.SHIPPED"
  capabilities:
    - consumerOf:
        - address: "ORDERS.PROCESSED"
          appName: "first-app"
          appNamespace: "service-app-project"
      producerOf:
        - address: "ORDERS.SHIPPED"
EOF
```

```bash {"stage":"deploy_app", "label":"wait for second-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp second-app -n service-app-project --for=condition=Ready --timeout=300s
```

#### third-app (Delivery Service)

```bash {"stage":"deploy_app", "label":"create third-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: third-app-app-cert
  namespace: service-app-project
spec:
  secretName: third-app-app-cert
  commonName: third-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_app", "label":"wait for third-app cert", "runtime":"bash"}
kubectl wait certificate third-app-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy third-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: third-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      forWorkQueue: "true"
  sharedAddresses:
    - address: "ORDERS.DELIVERED"
  capabilities:
    - consumerOf:
        - address: "ORDERS.SHIPPED"
          appName: "second-app"
          appNamespace: "service-app-project"
      producerOf:
        - address: "ORDERS.DELIVERED"
EOF
```

```bash {"stage":"deploy_app", "label":"wait for third-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp third-app -n service-app-project --for=condition=Ready --timeout=300s
```

#### master-sink (Optional operational drain)

`master-sink` is not part of the business processing pipeline. It is an optional operational drain that you enable when you want to consume messages accumulating on the terminal `ORDERS.DELIVERED` queue — for example, to prevent unbounded growth during a long-running demo, or as an explicit "pipeline complete" acknowledgement.

During the normal pipeline demonstration and the bottleneck/scale scenarios, keep this deployment at **0 replicas** so that `ORDERS.DELIVERED` depth remains visible in Grafana.

```bash {"stage":"deploy_app", "label":"create master-sink cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: master-sink-app-cert
  namespace: service-app-project
spec:
  secretName: master-sink-app-cert
  commonName: master-sink-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"deploy_app", "label":"wait for master-sink cert", "runtime":"bash"}
kubectl wait certificate master-sink-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy master-sink brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: master-sink-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      forWorkQueue: "true"
  capabilities:
    - consumerOf:
        - address: "ORDERS.DELIVERED"
          appName: "third-app"
          appNamespace: "service-app-project"
EOF
```

```bash {"stage":"deploy_app", "label":"wait for master-sink brokerapp", "runtime":"bash"}
kubectl wait BrokerApp master-sink-app -n service-app-project --for=condition=Ready --timeout=300s
```

The Camel Deployment for master-sink is deployed at **0 replicas**. Scale it up only when you want to actively drain `ORDERS.DELIVERED`.

```bash {"stage":"deploy_app", "label":"create master-sink pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-sink
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

```bash {"stage":"deploy_app", "label":"wait for master-sink binding secret", "runtime":"bash"}
kubectl wait secret master-sink-app-binding-secret -n service-app-project --for=create --timeout=300s
```

```bash {"stage":"deploy_app", "label":"deploy master-sink camel app", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: camel-jms-master-sink
  namespace: service-app-project
spec:
  replicas: 0
  selector:
    matchLabels:
      app: camel-jms-master-sink
  template:
    metadata:
      labels:
        app: camel-jms-master-sink
    spec:
      containers:
      - name: camel-jms-app
        image: quay.io/rh-ee-vnachiap/camel-jms-app:pipeline-1.0
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "100m"
        env:
        - name: BROKER_HOST
          valueFrom:
            secretKeyRef:
              name: master-sink-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: master-sink-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "master-sink-app"
        - name: APP_ROLE
          value: "sink"
        - name: CONSUMER_QUEUE
          value: "ORDERS.DELIVERED"
        - name: PRODUCER_QUEUE
          value: "NONE"
        - name: CONSUMER_CONCURRENCY
          value: "5"
        - name: JDK_JAVA_OPTIONS
          value: "-Xbootclasspath/a:/deployments/lib/main/de.dentrassi.crypto.pem-keystore-3.0.0.jar:/deployments/lib/main/com.hierynomus.asn-one-0.6.0.jar:/deployments/lib/main/org.slf4j.slf4j-api-2.0.18.jar -Djava.security.properties=/app/tls/pem/java.security"
        volumeMounts:
        - name: trust
          mountPath: /app/tls/ca
          readOnly: true
        - name: cert
          mountPath: /app/tls/client
          readOnly: true
        - name: pem
          mountPath: /app/tls/pem
          readOnly: true
      volumes:
      - name: trust
        secret:
          secretName: arkmq-org-broker-manager-ca
      - name: cert
        secret:
          secretName: master-sink-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-sink
EOF
```

To drain `ORDERS.DELIVERED` at any point during the tutorial, scale it up:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=1 -n service-app-project
```

To stop draining and let the queue accumulate again:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=0 -n service-app-project
```

### Wait for All Apps Provisioned

```bash {"stage":"deploy_app", "label":"wait for all apps provisioned", "runtime":"bash"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=AppsProvisioned --timeout=300s
kubectl wait pod --selector=ActiveMQArtemis=messaging-service -n service-app-project --for=condition=Ready --timeout=300s
```

### Verify BrokerApp Bindings

Each `BrokerApp` causes the Operator to create a binding secret containing the broker host and port for that application's dedicated acceptor. The Camel Deployments in the next section read these secrets directly — no manual connection string management required.

```bash {"stage":"deploy_app", "label":"verify brokerapp bindings", "runtime":"bash"}
kubectl get brokerapp -n service-app-project \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,PORT:.status.service.assignedPort,SECRET:.status.service.secret'
```

Expected output:

```
NAME              READY   PORT    SECRET
first-app         True    61617   first-app-binding-secret
order-generator   True    61618   order-generator-binding-secret
second-app        True    61619   second-app-binding-secret
third-app         True    61620   third-app-binding-secret
```

Then confirm the secrets exist:

```bash {"stage":"deploy_app", "label":"verify binding secrets", "runtime":"bash"}
kubectl get secret -n service-app-project | grep binding-secret
```

You should see `order-generator-binding-secret`, `first-app-binding-secret`, `second-app-binding-secret`, and `third-app-binding-secret` before proceeding to deploy the Camel applications.

---

## 5. Deploy Camel Applications

The same Docker image is deployed four times. Role and queue configuration are injected via environment variables — no rebuilding required.

### Shared PEM Secret

```bash {"stage":"deploy_camel", "label":"create pemcfg secret", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

Each deployment mounts its own app certificate at `/app/tls/client`. All four deployments share the same `cert-pemcfg` content (the PEM keystore type configuration), but mount different cert secrets for their individual mTLS identity.

### order-generator

Produces 25 realistic order JSON messages per second into `ORDERS.NEW`. The generator has its own `BrokerApp` identity (`order-generator`) with a single `producerOf: ORDERS.NEW` capability. This enforces least-privilege: `first-app` cannot send to `ORDERS.NEW` and `order-generator` cannot consume from it.

Wait for the `order-generator` binding secret before deploying:

```bash {"stage":"deploy_camel", "label":"wait for order-generator binding secret", "runtime":"bash"}
kubectl wait secret order-generator-binding-secret -n service-app-project --for=create --timeout=300s
```

Create the PEM keystore secret for the generator's own certificate:

```bash {"stage":"deploy_camel", "label":"create order-generator pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-generator
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

```bash {"stage":"deploy_camel", "label":"deploy order-generator", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-generator
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-generator
  template:
    metadata:
      labels:
        app: order-generator
    spec:
      containers:
      - name: camel-jms-app
        image: quay.io/rh-ee-vnachiap/camel-jms-app:pipeline-1.0
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "100m"
        env:
        - name: BROKER_HOST
          valueFrom:
            secretKeyRef:
              name: order-generator-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: order-generator-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "order-generator"
        - name: APP_ROLE
          value: "generator"
        - name: PRODUCER_QUEUE
          value: "ORDERS.NEW"
        - name: MESSAGE_RATE
          value: "5"
        - name: JDK_JAVA_OPTIONS
          value: "-Xbootclasspath/a:/deployments/lib/main/de.dentrassi.crypto.pem-keystore-3.0.0.jar:/deployments/lib/main/com.hierynomus.asn-one-0.6.0.jar:/deployments/lib/main/org.slf4j.slf4j-api-2.0.18.jar -Djava.security.properties=/app/tls/pem/java.security"
        volumeMounts:
        - name: trust
          mountPath: /app/tls/ca
          readOnly: true
        - name: cert
          mountPath: /app/tls/client
          readOnly: true
        - name: pem
          mountPath: /app/tls/pem
          readOnly: true
      volumes:
      - name: trust
        secret:
          secretName: arkmq-org-broker-manager-ca
      - name: cert
        secret:
          secretName: order-generator-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-generator
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for order-generator", "runtime":"bash"}
kubectl wait deployment order-generator -n service-app-project --for=condition=Available --timeout=300s
```

### camel-jms-app (Order Processor — first-app)

```bash {"stage":"deploy_camel", "label":"wait for first-app binding secret", "runtime":"bash"}
kubectl wait secret first-app-binding-secret -n service-app-project --for=create --timeout=300s
```

```bash {"stage":"deploy_camel", "label":"create first-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-first
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

```bash {"stage":"deploy_camel", "label":"deploy camel-jms-app", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: camel-jms-app
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: camel-jms-app
  template:
    metadata:
      labels:
        app: camel-jms-app
    spec:
      containers:
      - name: camel-jms-app
        image: quay.io/rh-ee-vnachiap/camel-jms-app:pipeline-1.0
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "100m"
        env:
        - name: BROKER_HOST
          valueFrom:
            secretKeyRef:
              name: first-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: first-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "first-app"
        - name: APP_ROLE
          value: "processor"
        - name: CONSUMER_QUEUE
          value: "ORDERS.NEW"
        - name: PRODUCER_QUEUE
          value: "ORDERS.PROCESSED"
        - name: PROCESSING_DELAY_MS
          value: "100"
        - name: CONSUMER_CONCURRENCY
          value: "1"
        # Throughput: 1 consumer / 0.1 s = ~10 msg/s — 2x headroom above the 5 msg/s generator rate.
        - name: JDK_JAVA_OPTIONS
          value: "-Xbootclasspath/a:/deployments/lib/main/de.dentrassi.crypto.pem-keystore-3.0.0.jar:/deployments/lib/main/com.hierynomus.asn-one-0.6.0.jar:/deployments/lib/main/org.slf4j.slf4j-api-2.0.18.jar -Djava.security.properties=/app/tls/pem/java.security"
        volumeMounts:
        - name: trust
          mountPath: /app/tls/ca
          readOnly: true
        - name: cert
          mountPath: /app/tls/client
          readOnly: true
        - name: pem
          mountPath: /app/tls/pem
          readOnly: true
      volumes:
      - name: trust
        secret:
          secretName: arkmq-org-broker-manager-ca
      - name: cert
        secret:
          secretName: first-app-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-first
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for camel-jms-app", "runtime":"bash"}
kubectl wait deployment camel-jms-app -n service-app-project --for=condition=Available --timeout=300s
```

### camel-jms-app-second (Shipping Service — second-app)

```bash {"stage":"deploy_camel", "label":"create second-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-second
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for second-app binding secret", "runtime":"bash"}
kubectl wait secret second-app-binding-secret -n service-app-project --for=create --timeout=300s
```

```bash {"stage":"deploy_camel", "label":"deploy camel-jms-app-second", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: camel-jms-app-second
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: camel-jms-app-second
  template:
    metadata:
      labels:
        app: camel-jms-app-second
    spec:
      containers:
      - name: camel-jms-app
        image: quay.io/rh-ee-vnachiap/camel-jms-app:pipeline-1.0
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "100m"
        env:
        - name: BROKER_HOST
          valueFrom:
            secretKeyRef:
              name: second-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: second-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "second-app"
        - name: APP_ROLE
          value: "shipping"
        - name: CONSUMER_QUEUE
          value: "ORDERS.PROCESSED"
        - name: PRODUCER_QUEUE
          value: "ORDERS.SHIPPED"
        - name: PROCESSING_DELAY_MS
          value: "25"
        - name: CONSUMER_CONCURRENCY
          value: "1"
        # Throughput: 1 consumer / 0.025 s = ~40 msg/s — 8x headroom above the 5 msg/s generator rate.
        - name: JDK_JAVA_OPTIONS
          value: "-Xbootclasspath/a:/deployments/lib/main/de.dentrassi.crypto.pem-keystore-3.0.0.jar:/deployments/lib/main/com.hierynomus.asn-one-0.6.0.jar:/deployments/lib/main/org.slf4j.slf4j-api-2.0.18.jar -Djava.security.properties=/app/tls/pem/java.security"
        volumeMounts:
        - name: trust
          mountPath: /app/tls/ca
          readOnly: true
        - name: cert
          mountPath: /app/tls/client
          readOnly: true
        - name: pem
          mountPath: /app/tls/pem
          readOnly: true
      volumes:
      - name: trust
        secret:
          secretName: arkmq-org-broker-manager-ca
      - name: cert
        secret:
          secretName: second-app-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-second
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for camel-jms-app-second", "runtime":"bash"}
kubectl rollout status deployment/camel-jms-app-second -n service-app-project --timeout=120s
kubectl wait deployment camel-jms-app-second -n service-app-project --for=condition=Available --timeout=300s
```

### camel-jms-app-third (Delivery Service — third-app)

```bash {"stage":"deploy_camel", "label":"create third-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-third
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for third-app binding secret", "runtime":"bash"}
kubectl wait secret third-app-binding-secret -n service-app-project --for=create --timeout=300s
```

```bash {"stage":"deploy_camel", "label":"deploy camel-jms-app-third", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: camel-jms-app-third
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: camel-jms-app-third
  template:
    metadata:
      labels:
        app: camel-jms-app-third
    spec:
      containers:
      - name: camel-jms-app
        image: quay.io/rh-ee-vnachiap/camel-jms-app:pipeline-1.0
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "100m"
        env:
        - name: BROKER_HOST
          valueFrom:
            secretKeyRef:
              name: third-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: third-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "third-app"
        - name: APP_ROLE
          value: "delivery"
        - name: CONSUMER_QUEUE
          value: "ORDERS.SHIPPED"
        - name: PRODUCER_QUEUE
          value: "ORDERS.DELIVERED"
        - name: PROCESSING_DELAY_MS
          value: "25"
        - name: CONSUMER_CONCURRENCY
          value: "1"
        # Throughput: 1 consumer / 0.025 s = ~40 msg/s — above the 25 msg/s generator rate.
        - name: JDK_JAVA_OPTIONS
          value: "-Xbootclasspath/a:/deployments/lib/main/de.dentrassi.crypto.pem-keystore-3.0.0.jar:/deployments/lib/main/com.hierynomus.asn-one-0.6.0.jar:/deployments/lib/main/org.slf4j.slf4j-api-2.0.18.jar -Djava.security.properties=/app/tls/pem/java.security"
        volumeMounts:
        - name: trust
          mountPath: /app/tls/ca
          readOnly: true
        - name: cert
          mountPath: /app/tls/client
          readOnly: true
        - name: pem
          mountPath: /app/tls/pem
          readOnly: true
      volumes:
      - name: trust
        secret:
          secretName: arkmq-org-broker-manager-ca
      - name: cert
        secret:
          secretName: third-app-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-third
EOF
```

```bash {"stage":"deploy_camel", "label":"wait for camel-jms-app-third", "runtime":"bash"}
kubectl rollout status deployment/camel-jms-app-third -n service-app-project --timeout=120s
kubectl wait deployment camel-jms-app-third -n service-app-project --for=condition=Available --timeout=300s
```

### Verify the Pipeline

Check that orders are flowing through all stages:

```bash {"stage":"verify", "label":"check generator logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/order-generator --tail=20
```

```bash {"stage":"verify", "label":"check processor logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/camel-jms-app --tail=20
```

```bash {"stage":"verify", "label":"check shipping logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/camel-jms-app-second --tail=20
```

```bash {"stage":"verify", "label":"check delivery logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/camel-jms-app-third --tail=20
```

You should see log lines like:

```
[generator]  → ORDERS.NEW      | orderId=ORD-8f31a2...
[processor]  ← ORDERS.NEW      | processing...
[processor]  → ORDERS.PROCESSED | status=PROCESSED
[shipping]   ← ORDERS.PROCESSED | shipping...
[shipping]   → ORDERS.SHIPPED   | status=SHIPPED
[delivery]   ← ORDERS.SHIPPED   | delivering...
[delivery]   → ORDERS.DELIVERED | status=DELIVERED
```

---

## 6. Configure Prometheus Monitoring

### Create Prometheus Client Certificate

**CRITICAL:** The certificate secret must be named exactly `prometheus-cert`. The ArkMQ Operator looks for this specific name to add the Prometheus identity to the broker's access control list.

```bash {"stage":"monitoring", "label":"create prometheus cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prometheus-cert
  namespace: service-app-project
spec:
  secretName: prometheus-cert
  commonName: prometheus
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```

```bash {"stage":"monitoring", "label":"wait for prometheus cert", "runtime":"bash"}
kubectl wait certificate prometheus-cert -n service-app-project --for=condition=Ready --timeout=300s
```

### Create Metrics Service

```bash {"stage":"monitoring", "label":"create metrics service", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: messaging-service-metrics
  namespace: service-app-project
  labels:
    app: messaging-service
spec:
  selector:
    ActiveMQArtemis: messaging-service
  ports:
    - name: metrics
      port: 8888
      targetPort: 8888
      protocol: TCP
EOF
```

### Create ServiceMonitor

```bash {"stage":"monitoring", "label":"set broker fqdn", "runtime":"bash"}
export BROKER_FQDN=messaging-service-ss-0.messaging-service-hdls-svc.service-app-project.svc.cluster.local
echo "Broker FQDN: ${BROKER_FQDN}"
```

```bash {"stage":"monitoring", "label":"create servicemonitor", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: messaging-service-monitor
  namespace: service-app-project
  labels:
    app: messaging-service
    release: prometheus
spec:
  selector:
    matchLabels:
      app: messaging-service
  endpoints:
  - port: metrics
    scheme: https
    interval: 15s
    tlsConfig:
      serverName: '${BROKER_FQDN}'
      ca:
        secret:
          name: arkmq-org-broker-manager-ca
          key: ca.pem
      cert:
        secret:
          name: prometheus-cert
          key: tls.crt
      keySecret:
        name: prometheus-cert
        key: tls.key
      insecureSkipVerify: false
EOF
```

The scrape interval is set to 15 seconds so the queue-depth panels update quickly during the operations scenarios.

### Create Prometheus Recording Rules

Pre-aggregated rules make the Grafana dashboard queries fast:

```bash {"stage":"monitoring", "label":"create recording rules", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: artemis-aggregation-rules
  namespace: service-app-project
  labels:
    release: prometheus
spec:
  groups:
  - name: artemis_aggregations
    interval: 15s
    rules:
    # Pipeline ingress rate — messages entering via ORDERS.NEW only (not double-counted across stages)
    - record: artemis:pipeline_ingress_rate
      expr: rate(broker_queue_messages_added_total{job="messaging-service-metrics",queue="ORDERS.NEW"}[1m])
    # Total messages currently waiting across all pipeline queues
    - record: artemis:pipeline_backlog
      expr: sum(broker_queue_message_count{job="messaging-service-metrics",queue=~"ORDERS[.].*"})
    # Total consumers across pipeline queues
    - record: artemis:pipeline_consumer_count
      expr: sum(broker_queue_consumer_count{job="messaging-service-metrics",queue=~"ORDERS[.].*"})
EOF
```

> **Troubleshooting metric names:** If your dashboard panels show "No data", the metric names exposed by your operator version may differ. Check what is available via Prometheus UI at http://localhost:9090 (after port-forwarding) by searching for `broker_queue`. Update the recording rule expressions and dashboard queries to match.

---

## 7. Create Grafana Dashboard

The dashboard is built around six panels that tell the operational story: throughput in, queue depth per stage, consumer count, and broker resources.

```bash {"stage":"grafana", "label":"create dashboard and apply via helm", "runtime":"bash"}
cat << 'EOF' > grafana-complete-values.yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      searchNamespace: ALL
    datasources:
      enabled: true
  dashboards:
    default:
      artemis-order-pipeline:
        json: |
          {
            "__inputs": [],
            "__requires": [],
            "annotations": { "list": [] },
            "editable": true,
            "gnetId": null,
            "graphTooltip": 0,
            "id": null,
            "links": [],
            "panels": [
              {
                "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
                "title": "Ingress Rate (ORDERS.NEW)",
                "description": "Messages entering the pipeline per second. Scoped to ORDERS.NEW to avoid double-counting across pipeline stages.",
                "type": "stat",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "artemis:pipeline_ingress_rate",
                    "refId": "A"
                  }
                ],
                "fieldConfig": {
                  "defaults": {
                    "unit": "short",
                    "color": { "mode": "thresholds" },
                    "thresholds": {
                      "mode": "absolute",
                      "steps": [
                        { "color": "green", "value": null },
                        { "color": "yellow", "value": 40 },
                        { "color": "red", "value": 80 }
                      ]
                    }
                  }
                }
              },
              {
                "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
                "title": "Total Messages Across Pipeline Queues",
                "description": "Sum of messages waiting in all ORDERS.* queues. A spike here means at least one stage is behind.",
                "type": "stat",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "artemis:pipeline_backlog",
                    "refId": "A"
                  }
                ],
                "fieldConfig": {
                  "defaults": {
                    "unit": "short",
                    "color": { "mode": "thresholds" },
                    "thresholds": {
                      "mode": "absolute",
                      "steps": [
                        { "color": "green", "value": null },
                        { "color": "yellow", "value": 100 },
                        { "color": "red", "value": 500 }
                      ]
                    }
                  }
                }
              },
              {
                "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 },
                "title": "Active Pipeline Consumers",
                "description": "Total JMS consumers connected across all ORDERS.* queues.",
                "type": "stat",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "artemis:pipeline_consumer_count",
                    "refId": "A"
                  }
                ],
                "fieldConfig": { "defaults": { "unit": "short" } }
              },
              {
                "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 },
                "title": "Broker Pod Memory",
                "description": "Kubernetes working set memory for the broker pod. Not the same as Artemis JVM heap.",
                "type": "stat",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "sum(container_memory_working_set_bytes{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\"})",
                    "refId": "A"
                  }
                ],
                "fieldConfig": { "defaults": { "unit": "bytes" } }
              },
              {
                "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
                "title": "Queue Depth per Stage",
                "description": "Watch ORDERS.PROCESSED grow when shipping is the bottleneck.",
                "type": "timeseries",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "broker_queue_message_count{job=\"messaging-service-metrics\",queue=~\"ORDERS[.].*\"}",
                    "legendFormat": "{{queue}}",
                    "refId": "A"
                  }
                ],
                "fieldConfig": {
                  "defaults": {
                    "unit": "short",
                    "custom": { "lineWidth": 2 }
                  }
                }
              },
              {
                "gridPos": { "h": 8, "w": 12, "x": 12, "y": 4 },
                "title": "Consumer Count per Queue",
                "description": "Scale camel-jms-app-second and watch the consumer count for ORDERS.PROCESSED increase.",
                "type": "timeseries",
                "datasource": { "type": "prometheus", "uid": "prometheus" },
                "targets": [
                  {
                    "expr": "broker_queue_consumer_count{job=\"messaging-service-metrics\",queue=~\"ORDERS[.].*\"}",
                    "legendFormat": "{{queue}}",
                    "refId": "A"
                  }
                ],
                "fieldConfig": { "defaults": { "unit": "short" } }
              }
            ],
            "refresh": "10s",
            "schemaVersion": 38,
            "style": "dark",
            "tags": ["artemis", "messaging", "pipeline"],
            "templating": { "list": [] },
            "time": { "from": "now-10m", "to": "now" },
            "timepicker": {},
            "timezone": "",
            "title": "Order Processing Pipeline",
            "uid": "order-pipeline",
            "version": 1,
            "weekStart": ""
          }
kubeControllerManager:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
EOF

helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n service-app-project \
  -f grafana-complete-values.yaml \
  --wait
```

```bash {"stage":"grafana", "label":"restart grafana", "runtime":"bash"}
kubectl rollout restart deployment/prometheus-grafana -n service-app-project
kubectl rollout status deployment/prometheus-grafana -n service-app-project --timeout=120s
```

### Access Grafana

```bash {"stage":"grafana", "label":"port forward grafana", "runtime":"bash"}
pkill -f "port-forward svc/prometheus-grafana" 2>/dev/null || true
sleep 1
kubectl port-forward svc/prometheus-grafana \
  -n service-app-project 3000:80 > /tmp/grafana-port-forward.log 2>&1 &
sleep 3
echo "Grafana available at http://localhost:3000"
```

```bash {"stage":"grafana", "label":"get grafana password", "runtime":"bash"}
kubectl get secret prometheus-grafana -n service-app-project -o jsonpath='{.data.admin-password}' | base64 -d && echo
```

Login with username `admin` and the password printed above, then open the **"Order Processing Pipeline"** dashboard.

Under normal conditions (5 msg/s, all consumers healthy) you should see:

| Panel | Expected value |
|---|---|
| Ingress Rate (ORDERS.NEW) | ~5 msg/s |
| Total Messages Across Pipeline Queues | ~0 (except ORDERS.DELIVERED — see below) |
| Active Pipeline Consumers | ~3 (see breakdown below) |
| Queue Depth per Stage | NEW/PROCESSED/SHIPPED flat near zero; DELIVERED growing |

**Why `ORDERS.DELIVERED` grows under normal conditions?** `master-sink` starts at **0 replicas** — there is deliberately no consumer on the terminal queue. `ORDERS.DELIVERED` will accumulate at ~5 msg/s. This is expected and useful: after ~20 seconds you should see a depth of roughly 100, proving the full pipeline is working end-to-end.

**Why ~3 consumers, not 4 applications?** The generator is producer-only (0 JMS consumers). `master-sink` starts at 0 replicas (0 consumers). Each processing stage opens one JMS session:

| Deployment | `CONSUMER_CONCURRENCY` | JMS consumers |
|---|---|---|
| camel-jms-app (`first-app`) | 1 | 1 |
| camel-jms-app-second (`second-app`) | 1 | 1 |
| camel-jms-app-third (`third-app`) | 1 | 1 |
| **Total** | | **3** |

> **Note:** This assumes all configured JMS sessions are connected. Verify the actual count in Prometheus by searching for `broker_queue_consumer_count` and inspecting the `queue` and `job` labels.

**Why the pipeline keeps up at baseline (theoretical capacities):**

| Stage | Delay | Concurrency | Theoretical max | Headroom vs 5 msg/s |
|---|---|---|---|---|
| processor | 100 ms | 1 | ~10 msg/s | 2x |
| shipping | 25 ms | 1 | ~40 msg/s | 8x |
| delivery | 25 ms | 1 | ~40 msg/s | 8x |

These are upper bounds assuming negligible JMS overhead. Every stage has comfortable headroom above the 5 msg/s generator rate.

---

## 8. Operations Scenarios

These three scenarios are the purpose of the tutorial. Each one creates or resolves a real operational event that you observe in Grafana.

### Scenario 1 — Normal Traffic

Everything is already running. Open the dashboard and confirm the pipeline is flowing at 5 msg/s.

```
Generator: 5 msg/s

processor (1x, 100 ms)  theoretical max ~10 msg/s  ->  ORDERS.PROCESSED  depth ~= 0
shipping  (1x,  25 ms)  theoretical max ~40 msg/s  ->  ORDERS.SHIPPED    depth ~= 0
delivery  (1x,  25 ms)  theoretical max ~40 msg/s  ->  ORDERS.DELIVERED  growing (no consumer)
```

**What to observe:** `ORDERS.NEW`, `ORDERS.PROCESSED`, and `ORDERS.SHIPPED` remain near zero. `ORDERS.DELIVERED` grows steadily because `master-sink` is intentionally disabled. Consumer count is stable (~3 at baseline — see consumer breakdown in the dashboard section above).

After ~20 seconds at 5 msg/s, `ORDERS.DELIVERED` should show a depth of roughly 100. That proves the full pipeline is flowing correctly end-to-end. To drain it at any point, scale up `master-sink`:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=1 -n service-app-project
```

### Scenario 2 — Create a Bottleneck

Increase the shipping processing delay to make it the bottleneck:

```bash {"stage":"scenario_bottleneck", "label":"slow down shipping", "runtime":"bash"}
kubectl set env deployment/camel-jms-app-second \
  PROCESSING_DELAY_MS=2000 \
  -n service-app-project
kubectl rollout status deployment/camel-jms-app-second -n service-app-project --timeout=120s
```

With `PROCESSING_DELAY_MS=2000` and `CONSUMER_CONCURRENCY=1`, the shipping service can process at most **0.5 msg/s** (theoretical). The generator is still producing 5 msg/s. Under idealized conditions `ORDERS.PROCESSED` should accumulate at roughly **4.5 messages per second** (5 − 0.5). The actual rate depends on JMS overhead and scheduling, but the growth will be clearly visible in Grafana within seconds.

**What to observe in Grafana:**

```
ORDERS.PROCESSED queue depth
        │
        │              ▲ growing
        │            ██
        │          ████
        │        ██████
        │      ████████
        └──────────────────► time
```

The `Queue Depth per Stage` panel shows `ORDERS.PROCESSED` climbing while the other queues stay flat. This is the bottleneck made visible.

### Scenario 3 — Scale to Recover

Scale the shipping deployment to five replicas:

```bash {"stage":"scenario_scale", "label":"scale up shipping", "runtime":"bash"}
kubectl scale deployment camel-jms-app-second \
  --replicas=5 \
  -n service-app-project
kubectl wait deployment camel-jms-app-second \
  -n service-app-project \
  --for=condition=Available \
  --timeout=300s
```

Scaling gives you five consumers, but they still have the 2-second processing delay — so aggregate throughput is still only ~2.5 msg/s at this point. The next step restores the 25 ms delay; after that rollout completes, aggregate theoretical capacity rises to ~200 msg/s, far exceeding the 5 msg/s input rate, and the backlog clears quickly:

```bash {"stage":"scenario_scale", "label":"reset shipping delay", "runtime":"bash"}
kubectl set env deployment/camel-jms-app-second \
  PROCESSING_DELAY_MS=25 \
  -n service-app-project
kubectl rollout status deployment/camel-jms-app-second -n service-app-project --timeout=120s
```

After the scale-up and delay reset, the updated consumer counts are:

| Queue | Consumers |
|---|---|
| `ORDERS.NEW` | 1 (unchanged) |
| `ORDERS.PROCESSED` | 5 (scaled up from 1) |
| `ORDERS.SHIPPED` | 1 (unchanged) |
| **Total** | **7** |

**What to observe in Grafana:**

```
ORDERS.PROCESSED queue depth
        |
        |      ^ grew during bottleneck
        |    ########
        |  ##########
        |  ######
        |  ####     <- draining after scale-up
        |  ##
        |  #
        |  0        <- recovered
        +-------------------> time
```

The `Consumer Count per Queue` panel shows `ORDERS.PROCESSED` consumer count jump from 1 to 5, and the `Total Messages Across Pipeline Queues` stat drop back toward zero.

**The operational story:**

> The shipping service became the bottleneck. We detected queue growth in Grafana and scaled the consumer deployment to restore throughput.

That is the complete demonstration of why BrokerService and real-time monitoring matter.

---

## Cleanup

```bash
# Stop port-forwarding
pkill -f "port-forward" 2>/dev/null || true

# Delete Camel deployments
kubectl delete deployment order-generator camel-jms-app camel-jms-app-second camel-jms-app-third -n service-app-project

# Delete Camel sink deployment (if it was scaled up)
kubectl delete deployment camel-jms-master-sink -n service-app-project 2>/dev/null || true

# Delete BrokerApps
kubectl delete BrokerApp order-generator first-app second-app third-app master-sink-app -n service-app-project

# Delete PEM config secrets
kubectl delete secret cert-pemcfg cert-pemcfg-generator cert-pemcfg-first cert-pemcfg-second cert-pemcfg-third cert-pemcfg-sink -n service-app-project

# Delete the BrokerService
kubectl delete BrokerService messaging-service -n service-app-project

# Delete monitoring resources
kubectl delete servicemonitor messaging-service-monitor -n service-app-project
kubectl delete service messaging-service-metrics -n service-app-project
kubectl delete prometheusrule artemis-aggregation-rules -n service-app-project

# Delete the namespace (removes everything remaining)
kubectl delete namespace service-app-project

# Delete the minikube cluster
minikube delete --profile brokerservice-monitoring
```

---

## Troubleshooting

### Metrics Not Appearing in Grafana

**1. Verify the broker pod is running:**
```bash
kubectl get pods -n service-app-project | grep messaging-service
```

**2. Check the broker metrics endpoint (manual, optional):**

The metrics endpoint on port 8888 uses **HTTPS/mTLS** — the same certificate chain
that Prometheus uses. A plain `curl localhost:8888/metrics` from inside the broker
container does **not** supply a client certificate and will therefore fail at the
TLS handshake. This command does **not** replicate the Prometheus scrape.

To verify that Prometheus can actually reach the broker, use the Prometheus Targets
page instead (see step 4 below). If you do need to make a manual mTLS request,
you must supply the Prometheus client certificate and the CA:

```bash
# From a pod that has the prometheus-cert secret mounted — for advanced debugging only.
# Normal tutorial flow does not require this.
curl --cacert /path/to/ca.pem \
     --cert /path/to/tls.crt \
     --key  /path/to/tls.key \
     https://messaging-service-ss-0.messaging-service-hdls-svc.service-app-project.svc.cluster.local:8888/metrics \
     | head -5
```

**3. Verify the ServiceMonitor has the correct label:**
```bash
kubectl get servicemonitor messaging-service-monitor -n service-app-project -o yaml | grep "release: prometheus"
```

**4. Check Prometheus targets (most useful first step):**
```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus \
  -n service-app-project 9090:9090 > /tmp/prometheus-pf.log 2>&1 &
```
Open http://localhost:9090/targets and verify `messaging-service-monitor` is `UP`.

If the target shows an error, the error message itself tells you whether the problem
is TLS, authentication, DNS, or network — far more useful than a manual curl.

**5. Verify the actual metric and job labels:**

The dashboard and recording rules assume `job="messaging-service-metrics"`. This label
comes from the `Service` name (`messaging-service-metrics`) via the ServiceMonitor.
If you renamed the Service, the job label will differ.

In Prometheus, search for `broker_queue_message_count` and inspect the returned labels.
The `job`, `instance`, and `queue` labels must match what the recording rules and
dashboard queries use. If `job` is different, update the recording rules and the
`job=` selectors in the Grafana dashboard queries accordingly.

**6. Verify the Grafana datasource UID:**

The dashboard JSON hard-codes `"uid": "prometheus"`. With `kube-prometheus-stack`
this is the default UID for the auto-provisioned Prometheus datasource, but it can
differ if the Helm chart was customised.

To verify:
```bash
# Port-forward Grafana if not already open
kubectl port-forward svc/prometheus-grafana -n service-app-project 3000:80 &
```
In Grafana, go to **Connections → Data Sources → Prometheus** and check the UID shown
in the URL (e.g. `/datasources/edit/prometheus`). If it differs from `prometheus`,
update the `"uid"` field in the dashboard JSON and re-import.

**7. Check Prometheus cert was found by the Operator:**
```bash
kubectl get secret prometheus-cert -n service-app-project
```
The Operator adds Prometheus to the broker JAAS allowlist only when a secret named
exactly `prometheus-cert` exists in the namespace.

### Pipeline Not Flowing

**Check that all four deployments are running:**
```bash
kubectl get deployment -n service-app-project
```

**Check for connection errors in any Camel pod:**
```bash
kubectl logs -n service-app-project deployment/camel-jms-app --tail=30 | grep -i error
```

**Verify binding secrets were created:**
```bash
kubectl get secret -n service-app-project | grep binding-secret
```
You should see `order-generator-binding-secret`, `first-app-binding-secret`, `second-app-binding-secret`, and `third-app-binding-secret`.
