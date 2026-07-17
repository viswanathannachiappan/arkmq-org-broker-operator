---
title: "Service and App CRD Round Trip"
description: "A tutorial on using BrokerService and BrokerApp CRDs, based on the 'round trip simple' e2e test."
draft: false
images: []
menu:
  docs:
    parent: "tutorials"
weight: 121
toc: true
---

This tutorial walks through a complete round trip of sending and receiving messages using the `BrokerService` and `BrokerApp` CRDs.

### Prerequisites

- A running Kubernetes cluster (this tutorial uses `minikube`).
- `kubectl` configured to interact with your cluster.

### 1. Setup

#### Start Minikube

```bash {"stage":"init", "id":"minikube_start", "runtime":"bash"}
minikube start --profile service-app-tutorial --extra-config=kubelet.sync-frequency=10s
minikube profile service-app-tutorial
```

#### Create Namespace

```bash {"stage":"init", "runtime":"bash" }
kubectl create namespace service-app-project
kubectl config set-context --current --namespace=service-app-project
```

#### Install Cert-Manager

```bash {"stage":"init", "label":"install cert-manager", "runtime":"bash"}
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.1/cert-manager.yaml
```

Wait for `cert-manager` to be ready.

```bash {"stage":"init", "label":"wait for cert-manager", "runtime":"bash"}
kubectl wait deployment --for=condition=Available -n cert-manager --timeout=600s cert-manager cert-manager-cainjector cert-manager-webhook
```

#### Install Trust Manager

First, add the Jetstack Helm repository.

```bash {"stage":"init", "label":"add jetstack helm repo", "runtime":"bash"}
helm repo add jetstack https://charts.jetstack.io --force-update
```

Now, install `trust-manager`.

```bash {"stage":"init", "label":"install trust-manager", "runtime":"bash"}
helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager --set secretTargets.enabled=true --set secretTargets.authorizedSecretsAll=true --wait
```

#### Install the Operator

```{"stage":"init", "rootdir":"$initial_dir"}
./deploy/install_opr.sh
```

Wait for the operator pod to become ready.

```bash {"stage":"init", "label":"wait for the operator to be running", "runtime":"bash"}
kubectl wait deployment arkmq-org-broker-controller-manager --for=create --timeout=240s
kubectl wait pod --all --for=condition=Ready --namespace=service-app-project --timeout=600s
```

### 2. Configure Certificates
We'll set up a CA and issue certificates for the operator, the service, and the application.

#### Create Issuers and Root Certificate

First the root issuer.

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

Then the root certificate.

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

Then a signing issuer that uses the root certificate.

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

#### Create Operator Certificate

##### Install the CA Bundle in the `cert-manager` namespace

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

##### Create the certificate for the operator

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

### 3. Deploy the Messaging Service and Application

#### Create Service Certificate

The service needs a certificate customized with a matching common name to enable
mTLS communication.

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

```bash {"stage":"deploy_certs", "label":"wait for operator cert", "runtime":"bash"}
kubectl wait certificate messaging-service-broker-cert -n service-app-project --for=condition=Ready --timeout=300s
```

#### Deploy `BrokerService`

```bash {"stage":"deploy_service", "label":"deploy service crd", "runtime":"bash"}
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
    limits:
      memory: "1Gi"
  env:
    - name: JAVA_ARGS_APPEND
      value: "-Dlog4j2.level=INFO"
EOF
```

Wait for the resource to be ready.

```bash {"stage":"deploy_service", "label":"wait for service"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=Ready --timeout=300s
```

#### Create Application Certificate

```bash {"stage":"deploy_app", "label":"create app cert", "runtime":"bash"}
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

#### Deploy `BrokerApp`

The `BrokerApp` connects to a `BrokerService` using label selectors and declares
its messaging capabilities. The operator automatically assigns a port from the
service's port pool for the application's acceptor.

```bash {"stage":"deploy_app", "label":"deploy app crd", "runtime":"bash"}
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
  capabilities:
    - producerOf:
        - address: "APP.JOBS"
      consumerOf:
        - address: "APP.JOBS"
EOF
```

Wait for the resource to be ready.

```bash {"stage":"deploy_app", "label":"wait for app", "runtime":"bash"}
kubectl wait BrokerApp first-app -n service-app-project --for=condition=Ready --timeout=300s
```

#### Verify Port Assignment

You can check the automatically assigned port in the app's status:

```bash {"stage":"deploy_app", "label":"check assigned port", "runtime":"bash"}
kubectl get BrokerApp first-app -n service-app-project -o jsonpath='{.status.service.assignedPort}'
```

### 4. Test Messaging

#### Create Client Configuration

```bash {"stage":"test_messaging", "label":"create pemcfg secret", "runtime":"bash"}
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

```bash {"stage":"test_messaging", "label":"wait for pemcfg secret", "runtime":"bash"}
until kubectl get secret cert-pemcfg -n service-app-project &> /dev/null; do echo "Waiting for secret..." && sleep 2; done
```

#### Deploy the Camel Load Driver Application

```bash {"stage":"test_messaging", "label":"deploy camel app", "runtime":"bash"}
cat <<'EOT' | kubectl apply -f -
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
        image: quay.io/rh-ee-vnachiap/camel-jms-app:4.0.0
        imagePullPolicy: Always
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
        - name: PRODUCER_QUEUE
          value: "APP.JOBS"
        - name: CONSUMER_QUEUE
          value: "APP.JOBS"
        - name: CLIENT_USERNAME
          value: "first-app"
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
          secretName: cert-pemcfg
EOT
```

```bash {"stage":"test_messaging", "label":"wait for camel app", "runtime":"bash"}
kubectl wait --for=condition=available --timeout=300s deployment/camel-jms-app -n service-app-project
```

#### Verify Application is Running

Check the pod logs to see messages being produced and consumed. You should see no connection errors and messages actively flowing.

```bash {"stage":"test_messaging", "label":"check logs", "runtime":"bash"}
kubectl logs -n service-app-project -l app=camel-jms-app --tail=50
```


Check the assigned port:

```bash {"stage":"test_messaging", "label":"check assigned port", "runtime":"bash"}
kubectl get BrokerApp first-app -n service-app-project -o jsonpath='{.status.service.assignedPort}'
```

View the binding secret contents:

```bash {"stage":"test_messaging", "label":"view binding secret", "runtime":"bash"}
kubectl get secret first-app-binding-secret -n service-app-project -o jsonpath='{.data.host}' | base64 -d && echo
kubectl get secret first-app-binding-secret -n service-app-project -o jsonpath='{.data.port}' | base64 -d && echo
```



### 5. Verify Application

Check the Camel application logs to see messages being produced and consumed:

```bash {"stage":"verify", "label":"check camel logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/camel-jms-app --tail=50
```

You should see:
- Producer route sending messages every 10 seconds
- Consumer route receiving and processing messages
- No connection errors


## 6. Monitoring BrokerService Deployments

### Important Note About Monitoring

**BrokerService does not expose Prometheus metrics by default.** BrokerService is designed as a simplified abstraction that hides technical details from application developers.

For full monitoring with Prometheus and Grafana, you should use the standard **Broker CR** instead, which:
- Exposes metrics on port 8888
- Supports ServiceMonitor configuration
- Provides detailed broker metrics (message rates, queue depth, etc.)

**See the [Prometheus Locked Down Tutorial](prometheus_locked_down.md)** for a complete monitoring setup with the Broker CR.

### Alternative: Monitor via kubectl

Platform operators can still monitor BrokerService deployments using kubectl commands:

**Check BrokerService status:**
```bash
kubectl get brokerservice messaging-service -n service-app-project -o yaml
```

**Check BrokerApp status:**
```bash
kubectl get brokerapp first-app -n service-app-project -o yaml
```

**Check broker pod logs:**
```bash
kubectl logs -n service-app-project messaging-service-ss-0 --tail=100
```

**Check broker pod resource usage:**
```bash
kubectl top pod messaging-service-ss-0 -n service-app-project
```


### Cleanup (Manual)

When you're finished exploring, clean up the resources:

```bash
# Delete the Camel application
kubectl delete deployment camel-jms-app -n service-app-project

# Delete the BrokerApp (this also deletes the binding secret)
kubectl delete BrokerApp first-app -n service-app-project

# Delete the BrokerService
kubectl delete BrokerService messaging-service -n service-app-project

# Delete the namespace (optional)
kubectl delete namespace service-app-project

# Delete the minikube cluster (optional)
minikube delete --profile service-app-tutorial
```
