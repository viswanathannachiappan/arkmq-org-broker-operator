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
                       order-processor-app
                          Order Processor
                                 │
                                 ▼
                       ORDERS.PROCESSED
                                 │
                       shipping-service-app
                          Shipping Service
                                 │
                                 ▼
                        ORDERS.SHIPPED
                                 │
                       delivery-service-app
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

**One reusable Camel image, four application roles.**
The same container image (`camel-jms-app`) is deployed four times for the core pipeline. An optional fifth deployment, `master-sink`, can drain the terminal queue when needed.
Role and queue configuration come from environment variables.

| Kubernetes Deployment | BrokerApp identity | `APP_ROLE` | Consumes | Produces |
|---|---|---|---|---|
| order-generator | `order-generator` | `generator` | — | `ORDERS.NEW` |
| order-processor-app | `order-processor-app` | `processor` | `ORDERS.NEW` | `ORDERS.PROCESSED` |
| shipping-service-app | `shipping-service-app` | `shipping` | `ORDERS.PROCESSED` | `ORDERS.SHIPPED` |
| delivery-service-app | `delivery-service-app` | `delivery` | `ORDERS.SHIPPED` | `ORDERS.DELIVERED` |

The Kubernetes Deployment name and BrokerApp identity are the same — the business service name — so every `kubectl` command and every Artemis access-control identity uses the same name throughout.

### Prerequisites

- A running Kubernetes cluster (this tutorial uses `minikube`)
- `kubectl` configured to interact with your cluster
- `helm` installed for deploying monitoring components
- A container build tool (`docker` or `podman`) available locally

> **Naming note:** Throughout this tutorial the optional terminal consumer is called `master-sink` as a conceptual role. The corresponding Kubernetes resources use more specific names: the BrokerApp is `master-sink-app`, and the Camel Deployment is `camel-jms-master-sink`.

---

## 2. Setup Infrastructure

### Start Minikube

```bash {"stage":"init", "id":"minikube_start", "runtime":"bash"}
minikube start \
  --profile brokerservice-monitoring \
  --cpus 2 \
  --memory 8192 \
  --disk-size 20000
minikube addons enable ingress --profile brokerservice-monitoring
```
```shell markdown_runner
* [brokerservice-monitoring] minikube v1.38.1 on Fedora 44
* Using the docker driver based on existing profile
* Starting "brokerservice-monitoring" primary control-plane node in "brokerservice-monitoring" cluster
* Pulling base image v0.0.50 ...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: default-storageclass, storage-provisioner
* Done! kubectl is now configured to use "brokerservice-monitoring" cluster and "default" namespace by default
* ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
  - Using image registry.k8s.io/ingress-nginx/controller:v1.14.3
  - Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.6.7
  - Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.6.7
* Verifying ingress addon...
* The 'ingress' addon is enabled
! You cannot change the memory size for an existing minikube cluster. Please first delete the cluster.
```

### Build the Camel Pipeline Image

The Camel pipeline image is built using your local Docker daemon (which has
internet access for Maven dependencies) and then loaded directly into Minikube.
The source lives alongside this tutorial in [`camel-jms-app/`](camel-jms-app/).
The `Containerfile` is a multi-stage build — Maven and the JDK run inside the
builder container, so no local JDK or Maven installation is required.

```bash {"stage":"init", "label":"build camel jms image", "rootdir":"$initial_dir", "runtime":"bash"}
docker build -f docs/tutorials/brokerservice/camel-jms-app/Containerfile docs/tutorials/brokerservice/camel-jms-app/ -t camel-jms-app:latest
minikube image load camel-jms-app:latest --profile brokerservice-monitoring
```
```shell markdown_runner
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Containerfile
#1 transferring dockerfile: 1.43kB done
#1 DONE 0.0s

#2 [internal] load metadata for registry.access.redhat.com/ubi9/openjdk-21:1.21
#2 DONE 0.6s

#3 [internal] load metadata for registry.access.redhat.com/ubi9/openjdk-25-runtime:1.24
#3 DONE 0.6s

#4 [internal] load .dockerignore
#4 transferring context: 217B done
#4 DONE 0.0s

#5 [stage-1 1/5] FROM registry.access.redhat.com/ubi9/openjdk-25-runtime:1.24@sha256:5d4ecb5d16665e601f3e7b3beea51801f8b87b57c2a526848b054104df314cb0
#5 resolve registry.access.redhat.com/ubi9/openjdk-25-runtime:1.24@sha256:5d4ecb5d16665e601f3e7b3beea51801f8b87b57c2a526848b054104df314cb0 0.1s done
#5 DONE 0.1s

#6 [builder 1/7] FROM registry.access.redhat.com/ubi9/openjdk-21:1.21@sha256:fa55b9f126da0d855e3709473e2237ad500f8d8e6a059f99b3ca10cdc2c5de58
#6 resolve registry.access.redhat.com/ubi9/openjdk-21:1.21@sha256:fa55b9f126da0d855e3709473e2237ad500f8d8e6a059f99b3ca10cdc2c5de58 0.1s done
#6 DONE 0.1s

#7 [internal] load build context
#7 transferring context: 890B done
#7 DONE 0.0s

#8 [builder 2/7] WORKDIR /build
#8 CACHED

#9 [builder 5/7] RUN mvn dependency:resolve-plugins dependency:resolve -q
#9 CACHED

#10 [stage-1 2/5] COPY --chown=185 --from=builder /build/target/quarkus-app/lib/       /deployments/lib/
#10 CACHED

#11 [stage-1 3/5] COPY --chown=185 --from=builder /build/target/quarkus-app/*.jar       /deployments/
#11 CACHED

#12 [stage-1 4/5] COPY --chown=185 --from=builder /build/target/quarkus-app/app/        /deployments/app/
#12 CACHED

#13 [builder 6/7] COPY src/ src/
#13 CACHED

#14 [builder 4/7] COPY pom.xml pom.xml
#14 CACHED

#15 [builder 7/7] RUN mvn package -DskipTests -q
#15 CACHED

#16 [builder 3/7] RUN microdnf install -y maven --setopt=install_weak_deps=0 && microdnf clean all
#16 CACHED

#17 [stage-1 5/5] COPY --chown=185 --from=builder /build/target/quarkus-app/quarkus/    /deployments/quarkus/
#17 CACHED

#18 exporting to image
#18 exporting layers done
#18 exporting manifest sha256:54e1acadb8a72fab196b6e0eb726d86f81108d1532095cb1e641e7d62a1669bf done
#18 exporting config sha256:54081b7726d6a8b49c24b5b9430b8163871ed77003e0be974611d6db0a8e9e57 done
#18 exporting attestation manifest sha256:b7236135ed9e853b86e53b48b3134de487f9753460f5279f9537013b9f5d74a3 0.0s done
#18 exporting manifest list sha256:e0ae805ebdc041bbde2333f7e04c6fe1127d6a0888b872cfeab18f5ef34683b2 0.0s done
#18 naming to docker.io/library/camel-jms-app:latest done
#18 unpacking to docker.io/library/camel-jms-app:latest 0.0s done
#18 DONE 0.1s
```

The first build takes a few minutes while Maven downloads dependencies and
compiles the Quarkus application. Subsequent builds reuse the cached dependency
layer and are much faster.

### Create Namespace

```bash {"stage":"init", "runtime":"bash"}
kubectl create namespace service-app-project
kubectl config set-context --current --namespace=service-app-project
```
```shell markdown_runner
namespace/service-app-project created
Context "brokerservice-monitoring" modified.
```

### Install Cert-Manager

```bash {"stage":"init", "label":"install cert-manager", "runtime":"bash"}
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.5/cert-manager.yaml
```
```shell markdown_runner
namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-cluster-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrole.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-tokenrequest created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-cert-manager-tokenrequest created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager-cainjector created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
```

Wait for `cert-manager` to be ready:

```bash {"stage":"init", "label":"wait for cert-manager", "runtime":"bash"}
kubectl wait deployment --for=condition=Available -n cert-manager --timeout=600s cert-manager cert-manager-cainjector cert-manager-webhook
```
```shell markdown_runner
deployment.apps/cert-manager condition met
deployment.apps/cert-manager-cainjector condition met
deployment.apps/cert-manager-webhook condition met
```

### Install Trust Manager

```bash {"stage":"init", "label":"add jetstack helm repo", "runtime":"bash"}
helm repo add jetstack https://charts.jetstack.io --force-update
```
```shell markdown_runner
"jetstack" has been added to your repositories
```

```bash {"stage":"init", "label":"install trust-manager", "runtime":"bash"}
helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager --set secretTargets.enabled=true --set secretTargets.authorizedSecretsAll=true --wait
```
```shell markdown_runner
Release "trust-manager" does not exist. Installing it now.
NAME: trust-manager
LAST DEPLOYED: Wed Sep  9 13:14:55 2026
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
⚠️  WARNING: Consider increasing the Helm value `replicaCount` to 2 if you require high availability.
⚠️  WARNING: Consider setting the Helm value `podDisruptionBudget.enabled` to true if you require high availability.

trust-manager v0.24.0 has been deployed successfully!
Your installation includes a default CA package, using the following
default CA package image:

:

It's imperative that you keep the default CA package image up to date.
To find out more about securely running trust-manager and to get started
with creating your first bundle, check out the documentation on the
cert-manager website:

https://cert-manager.io/docs/projects/trust-manager/
```

### Install kube-prometheus-stack

```bash {"stage":"init", "label":"add prometheus helm repo", "runtime":"bash"}
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```shell markdown_runner
"prometheus-community" already exists with the same configuration, skipping
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
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
```shell markdown_runner
Release "prometheus" does not exist. Installing it now.
NAME: prometheus
LAST DEPLOYED: Wed Sep  9 13:15:20 2026
NAMESPACE: service-app-project
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace service-app-project get pods -l "release=prometheus"

Get Grafana 'admin' user password by running:

  kubectl --namespace service-app-project get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace service-app-project get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus" -oname)
  kubectl --namespace service-app-project port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  kubectl get secret --namespace service-app-project -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

Wait for all monitoring components:

```bash {"stage":"init", "label":"wait for prometheus stack", "runtime":"bash"}
kubectl wait deployment --for=condition=Available -n service-app-project prometheus-grafana prometheus-kube-prometheus-operator --timeout=300s
kubectl wait statefulset --for=jsonpath='{.status.readyReplicas}'=1 -n service-app-project prometheus-prometheus-kube-prometheus-prometheus --timeout=300s
```
```shell markdown_runner
deployment.apps/prometheus-grafana condition met
deployment.apps/prometheus-kube-prometheus-operator condition met
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus condition met
```


### Set Up Grafana Dashboard

Grafana is already running (it was installed with kube-prometheus-stack above). Apply the dashboard ConfigMap now so the sidecar loads it immediately. Metrics will show "No data" until Section 6 configures the ServiceMonitor — that is expected.

```bash {"stage":"grafana", "label":"create dashboard configmap", "runtime":"bash"}
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: artemis-broker-health
  namespace: service-app-project
  labels:
    grafana_dashboard: "1"
data:
  artemis-broker-health.json: |
    {
      "title": "Artemis Broker - Memory & Queue Analysis",
      "uid": "artemis-broker-memory-queue-analysis",
      "style": "dark",
      "tags": [
        "artemis",
        "amq-broker",
        "jvm",
        "memory",
        "queue"
      ],
      "timezone": "",
      "editable": true,
      "graphTooltip": 1,
      "time": {
        "from": "now-30m",
        "to": "now"
      },
      "timepicker": {},
      "refresh": "5s",
      "schemaVersion": 38,
      "version": 34,
      "panels": [
        {
          "id": 200,
          "title": "Memory & Queue Performance",
          "type": "row",
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          }
        },
        {
          "id": 10,
          "title": "Container vs JVM vs Queue Memory",
          "description": "Memory comparison for the messaging-service Artemis broker.",
          "type": "timeseries",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 14,
            "w": 24,
            "x": 0,
            "y": 1
          },
          "targets": [
            {
              "expr": "sum(container_memory_working_set_bytes{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",container!=\"POD\",container!=\"\"})",
              "legendFormat": "Container Working Set",
              "refId": "A"
            },
            {
              "expr": "sum(jvm_memory_used_bytes{job=\"messaging-service-metrics\",area=\"heap\"})",
              "legendFormat": "JVM Heap Used",
              "refId": "B"
            },
            {
              "expr": "sum(broker_queue_persistent_size{job=\"messaging-service-metrics\"})",
              "legendFormat": "Queue Persistent Data",
              "refId": "C"
            },
            {
              "expr": "sum(kube_pod_container_resource_limits{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",resource=\"memory\"})",
              "legendFormat": "Container Memory Limit",
              "refId": "D"
            },
            {
              "expr": "sum(jvm_memory_max_bytes{job=\"messaging-service-metrics\",area=\"heap\"})",
              "legendFormat": "JVM Heap Max",
              "refId": "E"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "bytes",
              "min": 0,
              "custom": {
                "drawStyle": "line",
                "lineWidth": 2,
                "fillOpacity": 0,
                "showPoints": "never",
                "spanNulls": true,
                "axisPlacement": "left",
                "scaleDistribution": {
                  "type": "linear"
                }
              }
            },
            "overrides": [
              {
                "matcher": {
                  "id": "byFrameRefID",
                  "options": "A"
                },
                "properties": [
                  {
                    "id": "displayName",
                    "value": "Container Working Set"
                  },
                  {
                    "id": "color",
                    "value": {
                      "mode": "fixed",
                      "fixedColor": "purple"
                    }
                  },
                  {
                    "id": "custom.lineWidth",
                    "value": 3
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byFrameRefID",
                  "options": "B"
                },
                "properties": [
                  {
                    "id": "displayName",
                    "value": "JVM Heap Used"
                  },
                  {
                    "id": "color",
                    "value": {
                      "mode": "fixed",
                      "fixedColor": "orange"
                    }
                  },
                  {
                    "id": "custom.lineWidth",
                    "value": 2
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byFrameRefID",
                  "options": "C"
                },
                "properties": [
                  {
                    "id": "displayName",
                    "value": "Queue Persistent Data"
                  },
                  {
                    "id": "color",
                    "value": {
                      "mode": "fixed",
                      "fixedColor": "green"
                    }
                  },
                  {
                    "id": "custom.lineWidth",
                    "value": 2
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byFrameRefID",
                  "options": "D"
                },
                "properties": [
                  {
                    "id": "displayName",
                    "value": "Container Memory Limit"
                  },
                  {
                    "id": "color",
                    "value": {
                      "mode": "fixed",
                      "fixedColor": "blue"
                    }
                  },
                  {
                    "id": "custom.lineStyle",
                    "value": {
                      "fill": "dash",
                      "dash": [
                        8,
                        4
                      ]
                    }
                  },
                  {
                    "id": "custom.lineWidth",
                    "value": 2
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byFrameRefID",
                  "options": "E"
                },
                "properties": [
                  {
                    "id": "displayName",
                    "value": "JVM Heap Max"
                  },
                  {
                    "id": "color",
                    "value": {
                      "mode": "fixed",
                      "fixedColor": "red"
                    }
                  },
                  {
                    "id": "custom.lineStyle",
                    "value": {
                      "fill": "dash",
                      "dash": [
                        6,
                        4
                      ]
                    }
                  },
                  {
                    "id": "custom.lineWidth",
                    "value": 2
                  }
                ]
              }
            ]
          },
          "options": {
            "legend": {
              "displayMode": "table",
              "placement": "bottom",
              "calcs": [
                "lastNotNull",
                "max"
              ]
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          }
        },
        {
          "id": 300,
          "title": "Queue Storage Breakdown",
          "type": "row",
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 15
          }
        },
        {
          "id": 32,
          "title": "Current Queue Breakdown",
          "description": "Tabular view displaying persistent storage bytes and current message count for every Artemis queue.",
          "type": "table",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 10,
            "w": 24,
            "x": 0,
            "y": 16
          },
          "targets": [
            {
              "expr": "sum by (queue)(broker_queue_persistent_size{job=\"messaging-service-metrics\"})",
              "format": "table",
              "instant": true,
              "refId": "A"
            },
            {
              "expr": "sum by (queue)(broker_queue_message_count{job=\"messaging-service-metrics\"})",
              "format": "table",
              "instant": true,
              "refId": "B"
            }
          ],
          "transformations": [
            {
              "id": "merge",
              "options": {}
            },
            {
              "id": "organize",
              "options": {
                "excludeByName": {
                  "Time": true,
                  "Time 1": true,
                  "Time 2": true
                },
                "indexByName": {
                  "queue": 0,
                  "Value #A": 1,
                  "Value #B": 2
                },
                "renameByName": {
                  "queue": "Queue",
                  "Value #A": "Persistent Data",
                  "Value #B": "Messages"
                }
              }
            }
          ],
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "thresholds"
              },
              "custom": {
                "align": "auto"
              }
            },
            "overrides": [
              {
                "matcher": {
                  "id": "byName",
                  "options": "Queue"
                },
                "properties": [
                  {
                    "id": "custom.width",
                    "value": 260
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byName",
                  "options": "Persistent Data"
                },
                "properties": [
                  {
                    "id": "unit",
                    "value": "bytes"
                  },
                  {
                    "id": "custom.cellOptions",
                    "value": {
                      "type": "gauge",
                      "mode": "gradient"
                    }
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byName",
                  "options": "Messages"
                },
                "properties": [
                  {
                    "id": "unit",
                    "value": "short"
                  },
                  {
                    "id": "custom.cellOptions",
                    "value": {
                      "type": "auto"
                    }
                  }
                ]
              }
            ]
          },
          "options": {
            "showHeader": true,
            "cellHeight": "sm",
            "sortBy": [
              {
                "displayName": "Persistent Data",
                "desc": true
              }
            ]
          }
        }
      ]
    }
EOF
```
```shell markdown_runner
configmap/artemis-broker-health created
```

### Access Grafana

Create an Ingress to expose Grafana through the Minikube ingress controller (this tutorial uses the NGINX addon enabled at cluster start):

```bash {"stage":"grafana", "label":"create grafana ingress", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: service-app-project
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.brokerservice-monitoring.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
EOF
```
```shell markdown_runner
ingress.networking.k8s.io/grafana created
```

Add the Minikube IP to your `/etc/hosts` so the hostname resolves locally:

```bash {"stage":"grafana", "label":"configure hosts", "runtime":"bash"}
export CLUSTER_IP=$(minikube ip --profile brokerservice-monitoring)
echo "${CLUSTER_IP} grafana.brokerservice-monitoring.local" | sudo tee -a /etc/hosts
echo "Grafana available at http://grafana.brokerservice-monitoring.local"
```
```shell markdown_runner
192.168.49.2 grafana.brokerservice-monitoring.local
Grafana available at http://grafana.brokerservice-monitoring.local
```

```bash {"stage":"grafana", "label":"get grafana password", "runtime":"bash"}
kubectl get secret prometheus-grafana -n service-app-project -o jsonpath='{.data.admin-password}' | base64 -d && echo
```
```shell markdown_runner
0Iy5lsZlpqvwRZUl7nYIzZSk0lZ85KOMjyuw6Ugc
```

Login at **http://grafana.brokerservice-monitoring.local** with username `admin` and the password printed above, then open the **"Artemis Broker Operational Health & Performance"** dashboard.

Under normal conditions (5 msg/s, all consumers healthy) you should see:

| Panel | Expected value |
|---|---|
| Broker Pod Ready | `1` / HEALTHY |
| Broker Ready Replicas | `1` |
| Total Queue Messages | Low — dominated by `ORDERS.DELIVERED` which grows intentionally |
| Backlog With No Consumers | Reflects `ORDERS.DELIVERED` depth — growing while `master-sink` is disabled |
| Queue Message Count | `ORDERS.NEW`, `ORDERS.PROCESSED`, `ORDERS.SHIPPED` near zero; `ORDERS.DELIVERED` growing |
| Queue Consumer Count | ~1 consumer on each active processing queue |
| Messages Being Delivered | Activity visible while messages are in flight |
| Total Consumer Count | ~3 |
| Container CPU Usage | Low and stable |
| Container Memory Working Set % | Stable and below 70% |
| JVM Heap Utilization % | Stable and below 70% |

A growing `ORDERS.DELIVERED` queue is expected and does not indicate a pipeline failure — `master-sink` is intentionally disabled. Because `master-sink` starts at 0 replicas, `ORDERS.DELIVERED` accumulates at approximately 5 messages/sec. After about 20 seconds it should contain roughly 100 messages — a useful sanity check that the full pipeline is flowing end-to-end.

The expected baseline is three JMS consumers: one each for processor, shipping, and delivery. The generator is producer-only and `master-sink` starts with zero replicas.


> **Note:** The dashboard panels will show "No data" until the ServiceMonitor and broker metrics are configured in Section 6. The Grafana tab is useful to keep open so you can watch metrics appear as soon as the pipeline is running.

### Install the Operator

```bash {"stage":"init", "rootdir":"$initial_dir", "runtime":"bash"}
./deploy/install_opr.sh
```
```shell markdown_runner
Deploying operator to watch single namespace
Client Version: 4.8.11
Kubernetes Version: v1.35.1
customresourcedefinition.apiextensions.k8s.io/activemqartemises.broker.amq.io created
customresourcedefinition.apiextensions.k8s.io/activemqartemisaddresses.broker.amq.io created
customresourcedefinition.apiextensions.k8s.io/activemqartemisscaledowns.broker.amq.io created
customresourcedefinition.apiextensions.k8s.io/activemqartemissecurities.broker.amq.io created
customresourcedefinition.apiextensions.k8s.io/brokers.broker.arkmq.org created
customresourcedefinition.apiextensions.k8s.io/brokerapps.broker.arkmq.org created
customresourcedefinition.apiextensions.k8s.io/brokerclusters.broker.arkmq.org created
customresourcedefinition.apiextensions.k8s.io/brokerservices.broker.arkmq.org created
serviceaccount/arkmq-org-broker-controller-manager created
role.rbac.authorization.k8s.io/arkmq-org-broker-operator-role created
rolebinding.rbac.authorization.k8s.io/arkmq-org-broker-operator-rolebinding created
role.rbac.authorization.k8s.io/arkmq-org-broker-leader-election-role created
rolebinding.rbac.authorization.k8s.io/arkmq-org-broker-leader-election-rolebinding created
networkpolicy.networking.k8s.io/arkmq-org-broker-controller-manager-netpol created
deployment.apps/arkmq-org-broker-controller-manager created
```

```bash {"stage":"init", "label":"wait for the operator to be running", "runtime":"bash"}
kubectl wait deployment arkmq-org-broker-controller-manager --for=create --timeout=240s
kubectl wait pod --all --for=condition=Ready --namespace=service-app-project --timeout=600s
```
```shell markdown_runner
deployment.apps/arkmq-org-broker-controller-manager condition met
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0 condition met
pod/arkmq-org-broker-controller-manager-7c484b5568-xqhnz condition met
pod/prometheus-grafana-75f9888848-vzwmz condition met
pod/prometheus-kube-prometheus-operator-666bff8d55-tmqdv condition met
pod/prometheus-kube-state-metrics-8466c684bc-mkr2v condition met
pod/prometheus-prometheus-kube-prometheus-prometheus-0 condition met
pod/prometheus-prometheus-node-exporter-dn84n condition met
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
```shell markdown_runner
clusterissuer.cert-manager.io/root-issuer created
```

```bash {"stage":"deploy_certs", "label":"wait for root issuer", "runtime":"bash"}
kubectl wait clusterissuer root-issuer --for=condition=Ready --timeout=300s
```
```shell markdown_runner
clusterissuer.cert-manager.io/root-issuer condition met
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
```shell markdown_runner
certificate.cert-manager.io/root-cert created
```

```bash {"stage":"deploy_certs", "label":"wait for root cert", "runtime":"bash"}
kubectl wait certificate root-cert --for=condition=Ready -n cert-manager --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/root-cert condition met
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
```shell markdown_runner
clusterissuer.cert-manager.io/broker-ca-issuer created
```

```bash {"stage":"deploy_certs", "label":"wait for signing issuer", "runtime":"bash"}
kubectl wait clusterissuer broker-ca-issuer --for=condition=Ready --timeout=300s
```
```shell markdown_runner
clusterissuer.cert-manager.io/broker-ca-issuer condition met
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
```shell markdown_runner
bundle.trust.cert-manager.io/arkmq-org-broker-manager-ca created
```

```bash {"stage":"deploy_certs", "label":"wait for ca bundle", "runtime":"bash"}
kubectl wait bundle arkmq-org-broker-manager-ca -n cert-manager --for=condition=Synced --timeout=300s
```
```shell markdown_runner
bundle.trust.cert-manager.io/arkmq-org-broker-manager-ca condition met
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
```shell markdown_runner
certificate.cert-manager.io/arkmq-org-broker-manager-cert created
```

```bash {"stage":"deploy_certs", "label":"wait for operator cert", "runtime":"bash"}
kubectl wait certificate arkmq-org-broker-manager-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/arkmq-org-broker-manager-cert condition met
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
```shell markdown_runner
certificate.cert-manager.io/messaging-service-broker-cert created
```

```bash {"stage":"deploy_service", "label":"wait for broker cert", "runtime":"bash"}
kubectl wait certificate messaging-service-broker-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/messaging-service-broker-cert condition met
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
    app: "order-processing-pipeline"
spec:
  resources:
    limits:
      memory: "1Gi"
  env:
    - name: JAVA_ARGS_APPEND
      value: "-Dlog4j2.level=INFO"
EOF
```
```shell markdown_runner
brokerservice.broker.arkmq.org/messaging-service created
```
> The broker is configured with 1 GiB memory limit for this tutorial workload.


```bash {"stage":"deploy_service", "label":"wait for brokerservice", "runtime":"bash"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerservice.broker.arkmq.org/messaging-service condition met
```

### Deploy BrokerApps

Each `BrokerApp` declares exactly the permissions its pipeline stage needs. The ownership chain is:

```
order-generator  →  ORDERS.NEW  →  order-processor-app  →  ORDERS.PROCESSED  →  shipping-service-app  →  ORDERS.SHIPPED  →  delivery-service-app  →  ORDERS.DELIVERED
   (produce)              (consume/produce)                       (consume/produce)                              (consume/produce)
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
```shell markdown_runner
certificate.cert-manager.io/order-generator-app-cert created
```

```bash {"stage":"deploy_app", "label":"wait for order-generator cert", "runtime":"bash"}
kubectl wait certificate order-generator-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/order-generator-app-cert condition met
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
      app: "order-processing-pipeline"
  sharedAddresses:
    - address: "ORDERS.NEW"
  capabilities:
    - producerOf:
        - address: "ORDERS.NEW"
EOF
```
```shell markdown_runner
brokerapp.broker.arkmq.org/order-generator created
```

```bash {"stage":"deploy_app", "label":"wait for order-generator brokerapp", "runtime":"bash"}
kubectl wait BrokerApp order-generator -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerapp.broker.arkmq.org/order-generator condition met
```

#### order-processor-app (Order Processor)

```bash {"stage":"deploy_app", "label":"create order-processor-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: order-processor-app-cert
  namespace: service-app-project
spec:
  secretName: order-processor-app-cert
  commonName: order-processor-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```
```shell markdown_runner
certificate.cert-manager.io/order-processor-app-cert created
```

```bash {"stage":"deploy_app", "label":"wait for order-processor-app cert", "runtime":"bash"}
kubectl wait certificate order-processor-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/order-processor-app-cert condition met
```

```bash {"stage":"deploy_app", "label":"deploy order-processor-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: order-processor-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      app: "order-processing-pipeline"
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
```shell markdown_runner
brokerapp.broker.arkmq.org/order-processor-app created
```

```bash {"stage":"deploy_app", "label":"wait for order-processor-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp order-processor-app -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerapp.broker.arkmq.org/order-processor-app condition met
```

#### shipping-service-app (Shipping Service)

```bash {"stage":"deploy_app", "label":"create shipping-service-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: shipping-service-app-cert
  namespace: service-app-project
spec:
  secretName: shipping-service-app-cert
  commonName: shipping-service-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```
```shell markdown_runner
certificate.cert-manager.io/shipping-service-app-cert created
```

```bash {"stage":"deploy_app", "label":"wait for shipping-service-app cert", "runtime":"bash"}
kubectl wait certificate shipping-service-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/shipping-service-app-cert condition met
```

```bash {"stage":"deploy_app", "label":"deploy shipping-service-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: shipping-service-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      app: "order-processing-pipeline"
  sharedAddresses:
    - address: "ORDERS.SHIPPED"
  capabilities:
    - consumerOf:
        - address: "ORDERS.PROCESSED"
          appName: "order-processor-app"
          appNamespace: "service-app-project"
      producerOf:
        - address: "ORDERS.SHIPPED"
EOF
```
```shell markdown_runner
brokerapp.broker.arkmq.org/shipping-service-app created
```

```bash {"stage":"deploy_app", "label":"wait for shipping-service-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp shipping-service-app -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerapp.broker.arkmq.org/shipping-service-app condition met
```

#### delivery-service-app (Delivery Service)

```bash {"stage":"deploy_app", "label":"create delivery-service-app cert", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: delivery-service-app-cert
  namespace: service-app-project
spec:
  secretName: delivery-service-app-cert
  commonName: delivery-service-app
  issuerRef:
    name: broker-ca-issuer
    kind: ClusterIssuer
EOF
```
```shell markdown_runner
certificate.cert-manager.io/delivery-service-app-cert created
```

```bash {"stage":"deploy_app", "label":"wait for delivery-service-app cert", "runtime":"bash"}
kubectl wait certificate delivery-service-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/delivery-service-app-cert condition met
```

```bash {"stage":"deploy_app", "label":"deploy delivery-service-app brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: delivery-service-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      app: "order-processing-pipeline"
  sharedAddresses:
    - address: "ORDERS.DELIVERED"
  capabilities:
    - consumerOf:
        - address: "ORDERS.SHIPPED"
          appName: "shipping-service-app"
          appNamespace: "service-app-project"
      producerOf:
        - address: "ORDERS.DELIVERED"
EOF
```
```shell markdown_runner
brokerapp.broker.arkmq.org/delivery-service-app created
```

```bash {"stage":"deploy_app", "label":"wait for delivery-service-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp delivery-service-app -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerapp.broker.arkmq.org/delivery-service-app condition met
```

### Wait for All Apps Provisioned

```bash {"stage":"deploy_app", "label":"wait for all apps provisioned", "runtime":"bash"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=AppsProvisioned --timeout=300s
kubectl wait pod --selector=ActiveMQArtemis=messaging-service -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerservice.broker.arkmq.org/messaging-service condition met
pod/messaging-service-ss-0 condition met
```

### Verify BrokerApp Bindings

Each `BrokerApp` causes the Operator to create a binding secret containing the broker host and port for that application's dedicated acceptor. The Camel Deployments in the next section read these secrets directly — no manual connection string management required.

```bash {"stage":"deploy_app", "label":"verify brokerapp bindings", "runtime":"bash"}
kubectl get brokerapp -n service-app-project \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,PORT:.status.service.assignedPort,SECRET:.status.service.secret'
```
```shell markdown_runner
NAME                   READY   PORT    SECRET
delivery-service-app   True    61619   delivery-service-app-binding-secret
order-generator        True    61616   order-generator-binding-secret
order-processor-app    True    61617   order-processor-app-binding-secret
shipping-service-app   True    61618   shipping-service-app-binding-secret
```

Then confirm the secrets exist:

```bash {"stage":"deploy_app", "label":"verify binding secrets", "runtime":"bash"}
kubectl get secret -n service-app-project | grep binding-secret
```
```shell markdown_runner
delivery-service-app-binding-secret                                                   Opaque                                3      90s
order-generator-binding-secret                                                        Opaque                                3      5m3s
order-processor-app-binding-secret                                                    Opaque                                3      4m
shipping-service-app-binding-secret                                                   Opaque                                3      2m30s
```

You should see `order-generator-binding-secret`, `order-processor-app-binding-secret`, `shipping-service-app-binding-secret`, and `delivery-service-app-binding-secret` before proceeding to deploy the Camel applications.

---

## 5. Deploy Camel Applications

The same Docker image (`camel-jms-app`) is deployed four times — once for each pipeline stage. The role and queue configuration are provided through environment variables, so the image does not need to be rebuilt for each application.

The main configuration variables are:

| Variable | Purpose |
|---|---|
| `APP_ROLE` | Selects which Camel route the application runs |
| `CONSUMER_QUEUE` | Queue the application consumes from |
| `PRODUCER_QUEUE` | Queue the application produces to |
| `MESSAGE_RATE` | Messages per second for the generator |
| `PROCESSING_DELAY_MS` | Simulated processing delay per message |
| `ERROR_RATE` | Probability that processing a message fails |
| `CONSUMER_CONCURRENCY` | Number of concurrent JMS consumers per pod |

Each Camel deployment needs two things for mTLS:

- Its own **app certificate** (for example, `order-generator-app-cert`). cert-manager creates this certificate, and the Deployment mounts it at `/app/tls/client/`.
- A **PEM keystore configuration** that tells the [dentrassi PEM keystore](https://github.com/ctron/pem-keystore) library where to find the certificate and private key.

The PEM configuration is identical for all four applications because they all use the same filesystem paths:

- `/app/tls/client/tls.key`
- `/app/tls/client/tls.crt`

However, each Deployment mounts a different certificate Secret at that path. This means the applications share the same PEM configuration while retaining separate mTLS identities.

### PEM keystore configuration template

Create the PEM configuration once as a template. The same content is used for each application's `cert-pemcfg-*` Secret:

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
```shell markdown_runner
secret/cert-pemcfg created
```

> **Note:** The `cert-pemcfg-*` Secrets used by the individual Deployments contain this same configuration. The certificate itself is not stored in these Secrets — each application's certificate comes from its own cert-manager-generated Secret.

### order-generator

`order-generator` produces 5 order messages per second into `ORDERS.NEW`. The rate is controlled by the `MESSAGE_RATE` environment variable in the Camel Deployment. The BrokerApp only declares the messaging capability (`producerOf: ORDERS.NEW`).

To change the message rate, update the Deployment's `MESSAGE_RATE` value — no image rebuild is required.

Before deploying the application, wait for the binding Secret created for its BrokerApp:

```bash {"stage":"deploy_camel", "label":"wait for order-generator binding secret", "runtime":"bash"}
kubectl wait secret order-generator-binding-secret -n service-app-project --for=create --timeout=300s
```
```shell markdown_runner
secret/order-generator-binding-secret condition met
```

Create the PEM keystore configuration Secret for this application:

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
```shell markdown_runner
secret/cert-pemcfg-generator created
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
        image: camel-jms-app:latest
        imagePullPolicy: Never
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
```shell markdown_runner
deployment.apps/order-generator created
```

```bash {"stage":"deploy_camel", "label":"wait for order-generator", "runtime":"bash"}
kubectl wait deployment order-generator -n service-app-project --for=condition=Available --timeout=300s
```
```shell markdown_runner
deployment.apps/order-generator condition met
```

### order-processor-app (Order Processor)

```bash {"stage":"deploy_camel", "label":"wait for order-processor-app binding secret", "runtime":"bash"}
kubectl wait secret order-processor-app-binding-secret -n service-app-project --for=create --timeout=300s
```
```shell markdown_runner
secret/order-processor-app-binding-secret condition met
```

```bash {"stage":"deploy_camel", "label":"create order-processor-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-order-processor
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```
```shell markdown_runner
secret/cert-pemcfg-order-processor created
```

```bash {"stage":"deploy_camel", "label":"deploy order-processor-app", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor-app
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-processor-app
  template:
    metadata:
      labels:
        app: order-processor-app
    spec:
      containers:
      - name: camel-jms-app
        image: camel-jms-app:latest
        imagePullPolicy: Never
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
              name: order-processor-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: order-processor-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "order-processor-app"
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
          secretName: order-processor-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-order-processor
EOF
```
```shell markdown_runner
deployment.apps/order-processor-app created
```

```bash {"stage":"deploy_camel", "label":"wait for order-processor-app", "runtime":"bash"}
kubectl wait deployment order-processor-app -n service-app-project --for=condition=Available --timeout=300s
```
```shell markdown_runner
deployment.apps/order-processor-app condition met
```

### shipping-service-app (Shipping Service)

```bash {"stage":"deploy_camel", "label":"create shipping-service-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-shipping-service
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```
```shell markdown_runner
secret/cert-pemcfg-shipping-service created
```

```bash {"stage":"deploy_camel", "label":"wait for shipping-service-app binding secret", "runtime":"bash"}
kubectl wait secret shipping-service-app-binding-secret -n service-app-project --for=create --timeout=300s
```
```shell markdown_runner
secret/shipping-service-app-binding-secret condition met
```

```bash {"stage":"deploy_camel", "label":"deploy shipping-service-app", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping-service-app
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shipping-service-app
  template:
    metadata:
      labels:
        app: shipping-service-app
    spec:
      containers:
      - name: camel-jms-app
        image: camel-jms-app:latest
        imagePullPolicy: Never
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
              name: shipping-service-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: shipping-service-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "shipping-service-app"
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
          secretName: shipping-service-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-shipping-service
EOF
```
```shell markdown_runner
deployment.apps/shipping-service-app created
```

```bash {"stage":"deploy_camel", "label":"wait for shipping-service-app", "runtime":"bash"}
kubectl wait deployment shipping-service-app -n service-app-project --for=condition=Available --timeout=300s
```
```shell markdown_runner
deployment.apps/shipping-service-app condition met
```

### delivery-service-app (Delivery Service)

```bash {"stage":"deploy_camel", "label":"create delivery-service-app pemcfg", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cert-pemcfg-delivery-service
  namespace: service-app-project
type: Opaque
stringData:
  tls.pemcfg: |
    source.key=/app/tls/client/tls.key
    source.cert=/app/tls/client/tls.crt
  java.security: security.provider.6=de.dentrassi.crypto.pem.PemKeyStoreProvider
EOF
```
```shell markdown_runner
secret/cert-pemcfg-delivery-service created
```

```bash {"stage":"deploy_camel", "label":"wait for delivery-service-app binding secret", "runtime":"bash"}
kubectl wait secret delivery-service-app-binding-secret -n service-app-project --for=create --timeout=300s
```
```shell markdown_runner
secret/delivery-service-app-binding-secret condition met
```

```bash {"stage":"deploy_camel", "label":"deploy delivery-service-app", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery-service-app
  namespace: service-app-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: delivery-service-app
  template:
    metadata:
      labels:
        app: delivery-service-app
    spec:
      containers:
      - name: camel-jms-app
        image: camel-jms-app:latest
        imagePullPolicy: Never
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
              name: delivery-service-app-binding-secret
              key: host
        - name: BROKER_PORT
          valueFrom:
            secretKeyRef:
              name: delivery-service-app-binding-secret
              key: port
        - name: CLIENT_USERNAME
          value: "delivery-service-app"
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
        # Throughput: 1 consumer / 0.025 s = ~40 msg/s — 8x headroom above the 5 msg/s generator rate.
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
          secretName: delivery-service-app-cert
      - name: pem
        secret:
          secretName: cert-pemcfg-delivery-service
EOF
```
```shell markdown_runner
deployment.apps/delivery-service-app created
```

```bash {"stage":"deploy_camel", "label":"wait for delivery-service-app", "runtime":"bash"}
kubectl wait deployment delivery-service-app -n service-app-project --for=condition=Available --timeout=300s
```
```shell markdown_runner
deployment.apps/delivery-service-app condition met
```

### master-sink (Optional operational drain)

`master-sink` is not part of the business processing pipeline. It is an optional operational drain that you enable when you want to consume messages accumulating on the terminal `ORDERS.DELIVERED` queue — for example, to prevent unbounded growth during a long-running demo, or as an explicit "pipeline complete" acknowledgement.

During the normal pipeline demonstration and the bottleneck/scale scenarios, keep this deployment at **0 replicas** so that `ORDERS.DELIVERED` depth remains visible in Grafana.

```bash {"stage":"deploy_camel", "label":"create master-sink cert", "runtime":"bash"}
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
```shell markdown_runner
certificate.cert-manager.io/master-sink-app-cert created
```

```bash {"stage":"deploy_camel", "label":"wait for master-sink cert", "runtime":"bash"}
kubectl wait certificate master-sink-app-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/master-sink-app-cert condition met
```

```bash {"stage":"deploy_camel", "label":"deploy master-sink brokerapp", "runtime":"bash"}
kubectl apply -f - <<EOF
apiVersion: broker.arkmq.org/v1beta2
kind: BrokerApp
metadata:
  name: master-sink-app
  namespace: service-app-project
spec:
  selector:
    matchLabels:
      app: "order-processing-pipeline"
  capabilities:
    - consumerOf:
        - address: "ORDERS.DELIVERED"
          appName: "delivery-service-app"
          appNamespace: "service-app-project"
EOF
```
```shell markdown_runner
brokerapp.broker.arkmq.org/master-sink-app created
```

```bash {"stage":"deploy_camel", "label":"wait for master-sink brokerapp", "runtime":"bash"}
kubectl wait BrokerApp master-sink-app -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
brokerapp.broker.arkmq.org/master-sink-app condition met
```

The Camel JMS app uses the [dentrassi PEM keystore](https://github.com/ctron/pem-keystore) library to handle mTLS. The `master-sink-app-cert` Secret contains the TLS certificate and private key, which are mounted in the container at `/app/tls/client/`.

The `cert-pemcfg-sink` Secret provides the configuration needed to use those files:

- **`tls.pemcfg`** — tells the PEM library where to find the certificate and private key: `/app/tls/client/tls.crt` and `/app/tls/client/tls.key`.
- **`java.security`** — registers the PEM keystore provider with the JVM.

The paths in `tls.pemcfg` must match the certificate's `mountPath` in the Deployment.

```bash {"stage":"deploy_camel", "label":"create master-sink pemcfg", "runtime":"bash"}
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
```shell markdown_runner
secret/cert-pemcfg-sink created
```

```bash {"stage":"deploy_camel", "label":"wait for master-sink binding secret", "runtime":"bash"}
kubectl wait secret master-sink-app-binding-secret -n service-app-project --for=create --timeout=300s
```
```shell markdown_runner
secret/master-sink-app-binding-secret condition met
```

```bash {"stage":"deploy_camel", "label":"deploy master-sink camel app", "runtime":"bash"}
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
        image: camel-jms-app:latest
        imagePullPolicy: Never
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
```shell markdown_runner
deployment.apps/camel-jms-master-sink created
```

The Camel Deployment for master-sink is deployed at **0 replicas**. Scale it up only when you want to actively drain `ORDERS.DELIVERED`.

To drain `ORDERS.DELIVERED` at any point during the tutorial, scale it up:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=1 -n service-app-project
```

To stop draining and let the queue accumulate again:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=0 -n service-app-project
```

### Verify the Pipeline

Check that orders are flowing through all stages:

```bash {"stage":"verify", "label":"check generator logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/order-generator --tail=20
```
```shell markdown_runner
2026-09-09 12:25:01,883 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-3243fe
2026-09-09 12:25:01,921 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(372):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:5029634b-c29a-4dbf-ae17-2d50efcc6335:372 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:02,082 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-4a372a
2026-09-09 12:25:02,108 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(373):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:834485bf-a7c3-4ed7-aa6d-366da27e2219:373 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:02,283 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-f35ff8
2026-09-09 12:25:02,306 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(374):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:d5149c00-6ca3-41c3-a23f-42ae77b6a883:374 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:02,482 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-dac55f
2026-09-09 12:25:02,506 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(375):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:d5c078c1-4edd-419b-8e21-d75fa140e78d:375 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:02,683 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-702533
2026-09-09 12:25:02,707 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(376):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:9a12e3ac-9105-4345-8c49-ccb850d50e2d:376 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:02,883 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-1becca
2026-09-09 12:25:02,908 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(377):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:f1ea8635-66e0-4c3b-a981-b09c128a2181:377 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:03,083 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-edf661
2026-09-09 12:25:03,108 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(378):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:6e76e5cf-d4b7-46c3-8e05-125f65f101d1:378 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:03,282 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-57fe3c
2026-09-09 12:25:03,312 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(379):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:caa97e55-5de4-4d42-9d4f-3a0cd1aaa5fc:379 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:03,483 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-3bf97d
2026-09-09 12:25:03,509 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(380):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:ab30e060-efd1-464c-8f5d-9ea194a283c3:380 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
2026-09-09 12:25:03,683 INFO  [order-generator] (Camel (camel-1) thread #1 - timer://order-generator) [generator] → ORDERS.NEW | orderId=ORD-22173f
2026-09-09 12:25:03,721 INFO  [org.apache.qpid.jms.JmsConnection] (AmqpProvider :(381):[amqps://messaging-service.service-app-project.svc.cluster.local:61616]) Connection ID:d67f5897-f1e3-4c08-aa70-375e53f019b1:381 connected to server: amqps://messaging-service.service-app-project.svc.cluster.local:61616
```

```bash {"stage":"verify", "label":"check processor logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/order-processor-app --tail=20
```
```shell markdown_runner
2026-09-09 12:25:01,933 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:02,033 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:02,125 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:02,226 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:02,315 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:02,415 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:02,515 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:02,615 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:02,719 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:02,819 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:02,917 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:03,018 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:03,117 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:03,218 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:03,325 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:03,425 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:03,518 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:03,619 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
2026-09-09 12:25:03,732 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] ← ORDERS.NEW | processing...
2026-09-09 12:25:03,832 INFO  [order-processor] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.NEW]) [processor] → ORDERS.PROCESSED | status=PROCESSED
```

```bash {"stage":"verify", "label":"check shipping logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/shipping-service-app --tail=20
```
```shell markdown_runner
2026-09-09 12:25:02,043 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:02,068 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:02,235 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:02,260 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:02,425 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:02,451 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:02,627 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:02,652 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:02,828 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:02,853 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:03,028 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:03,054 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:03,228 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:03,254 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:03,435 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:03,461 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:03,631 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:03,657 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
2026-09-09 12:25:03,844 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] ← ORDERS.PROCESSED | shipping...
2026-09-09 12:25:03,869 INFO  [order-shipping] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.PROCESSED]) [shipping] → ORDERS.SHIPPED | status=SHIPPED
```

```bash {"stage":"verify", "label":"check delivery logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/delivery-service-app --tail=20
```
```shell markdown_runner
2026-09-09 12:25:02,081 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:02,106 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:02,268 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:02,294 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:02,462 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:02,488 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:02,662 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:02,688 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:02,861 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:02,886 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:03,064 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:03,090 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:03,265 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:03,290 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:03,471 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:03,497 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:03,667 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:03,692 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
2026-09-09 12:25:03,879 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] ← ORDERS.SHIPPED | delivering...
2026-09-09 12:25:03,904 INFO  [order-delivery] (Camel (camel-1) thread #1 - JmsConsumer[ORDERS.SHIPPED]) [delivery] → ORDERS.DELIVERED | status=DELIVERED
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

> **How broker metrics work:** The ArkMQ Operator automatically configures the Prometheus Java agent for every `BrokerService`, exposing broker metrics on port 8888. This is a `BrokerService`-level concern — individual `BrokerApp` resources do not configure metrics. This section creates a Kubernetes `Service` for the metrics port and a `ServiceMonitor` so Prometheus can discover and scrape it.

### Create Prometheus Client Certificate

Create the Prometheus client certificate. The Operator uses `prometheus-cert` as the default Prometheus client certificate secret name; it uses the certificate's Common Name to grant Prometheus access to the broker metrics endpoint.

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
```shell markdown_runner
certificate.cert-manager.io/prometheus-cert created
```

```bash {"stage":"monitoring", "label":"wait for prometheus cert", "runtime":"bash"}
kubectl wait certificate prometheus-cert -n service-app-project --for=condition=Ready --timeout=300s
```
```shell markdown_runner
certificate.cert-manager.io/prometheus-cert condition met
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
```shell markdown_runner
service/messaging-service-metrics created
```

### Create ServiceMonitor

```bash {"stage":"monitoring", "label":"set broker fqdn", "runtime":"bash"}
export BROKER_POD=$(kubectl get pods \
  -n service-app-project \
  -l ActiveMQArtemis=messaging-service \
  -o jsonpath='{.items[0].metadata.name}')
export BROKER_FQDN="${BROKER_POD}.messaging-service-hdls-svc.service-app-project.svc.cluster.local"
echo "Broker FQDN: ${BROKER_FQDN}"
```
```shell markdown_runner
Broker FQDN: messaging-service-ss-0.messaging-service-hdls-svc.service-app-project.svc.cluster.local
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
```shell markdown_runner
servicemonitor.monitoring.coreos.com/messaging-service-monitor created
```

### Create Prometheus Recording Rules

Create a recording rule for the total number of Artemis queue consumers. This keeps the dashboard query simple and provides a stable metric for the "Total Consumer Count" panel:

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
    # Total consumer count across all queues — used by the dashboard "Total Consumer Count" panel
    - record: artemis:total_consumer_count
      expr: sum(broker_queue_consumer_count{job="messaging-service-metrics"})
EOF
```
```shell markdown_runner
prometheusrule.monitoring.coreos.com/artemis-aggregation-rules created
```

---

## 7. Grafana Dashboard Reference

The Grafana dashboard was provisioned in Section 2 and is already accessible. This section describes what each panel shows.

The dashboard is organized into 4 rows:

| Row | Panels | What to look for |
|---|---|---|
| **Memory Overview** | Container Memory %, JVM Heap %, Queue Persistent Data, Container Memory Headroom | Threshold colouring: green < 70 %, yellow 70–85 %, red > 85 % |
| **Container, JVM & Queue Memory** | Single timeseries: Container Working Set (purple), JVM Heap Used (orange), Queue Persistent Data (green), Container Memory Limit (blue dashed) | All four lines share one left Y-axis in bytes. Queue Persistent Data is a broker storage metric — it does not directly equal JVM heap usage. |
| **Queue Storage Contribution** | Horizontal bar chart (current snapshot) and table with Persistent Data, Messages, % of Total | Identify which queues are retaining the most persistent data right now. |
| **JVM Diagnostics** | GC collection rate: Young GC (green), Old GC (red) | Elevated Old GC rate together with high JVM heap % can indicate memory pressure. |

## 8. Operations Scenarios

These three scenarios are the purpose of the tutorial. Each one creates or resolves a real operational event that you observe in Grafana.

### Scenario 1 — Normal Traffic

Everything is already running. Open the dashboard and confirm the pipeline is flowing at 5 msg/s.

Each stage has comfortable headroom at the baseline rate:

| Stage | Delay | Consumers | Approx. capacity |
|---|---|---|---|
| processor | 100 ms | 1 | ~10 msg/s |
| shipping | 25 ms | 1 | ~40 msg/s |
| delivery | 25 ms | 1 | ~40 msg/s |

**What to observe:** `ORDERS.NEW`, `ORDERS.PROCESSED`, and `ORDERS.SHIPPED` remain near zero. `ORDERS.DELIVERED` grows steadily because `master-sink` is intentionally disabled. To drain it at any point:

```bash
kubectl scale deployment camel-jms-master-sink --replicas=1 -n service-app-project
```

### Scenario 2 — Create a Bottleneck

Increase the shipping processing delay to make it the bottleneck:

```bash {"stage":"scenario_bottleneck", "label":"slow down shipping", "runtime":"bash"}
kubectl set env deployment/shipping-service-app \
  PROCESSING_DELAY_MS=2000 \
  -n service-app-project
kubectl rollout status deployment/shipping-service-app -n service-app-project --timeout=120s
```
```shell markdown_runner
deployment.apps/shipping-service-app env updated
Waiting for deployment "shipping-service-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "shipping-service-app" rollout to finish: 1 old replicas are pending termination...
deployment "shipping-service-app" successfully rolled out
```

With `PROCESSING_DELAY_MS=2000` and `CONSUMER_CONCURRENCY=1`, the shipping service can process at most **0.5 msg/s** (theoretical). The generator is still producing 5 msg/s, so under idealized conditions `ORDERS.PROCESSED` should accumulate at roughly **4.5 messages per second** (5 − 0.5). The actual rate depends on JMS overhead and scheduling, but the growth will be clearly visible in Grafana within seconds.

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

The `Queue Message Count` panel shows `ORDERS.PROCESSED` climbing while the other queues stay flat. This is the bottleneck made visible.

### Scenario 3 — Scale to Recover

Scale the shipping deployment to five replicas:

```bash {"stage":"scenario_scale", "label":"scale up shipping", "runtime":"bash"}
kubectl scale deployment shipping-service-app \
  --replicas=5 \
  -n service-app-project
kubectl wait deployment shipping-service-app \
  -n service-app-project \
  --for=condition=Available \
  --timeout=300s
```
```shell markdown_runner
deployment.apps/shipping-service-app scaled
deployment.apps/shipping-service-app condition met
```

Scaling gives you five consumers, but they still have the 2-second processing delay — so aggregate throughput is still only ~2.5 msg/s at this point. The backlog may therefore continue growing during the rollout. The next step restores the 25 ms delay; after that rollout completes, aggregate theoretical capacity rises to ~200 msg/s — though actual throughput will be lower due to JMS and scheduling overhead. With the generator still producing 5 msg/s, the backlog can drain at up to roughly 195 msg/s under idealized conditions, so even a large accumulated backlog clears quickly:

```bash {"stage":"scenario_scale", "label":"reset shipping delay", "runtime":"bash"}
kubectl set env deployment/shipping-service-app \
  PROCESSING_DELAY_MS=25 \
  -n service-app-project
kubectl rollout status deployment/shipping-service-app -n service-app-project --timeout=120s
```
```shell markdown_runner
deployment.apps/shipping-service-app env updated
Waiting for deployment "shipping-service-app" rollout to finish: 0 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 0 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "shipping-service-app" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "shipping-service-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "shipping-service-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "shipping-service-app" rollout to finish: 1 old replicas are pending termination...
deployment "shipping-service-app" successfully rolled out
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

Because `CONSUMER_CONCURRENCY=1`, the five shipping replicas create five JMS consumers on `ORDERS.PROCESSED`. The `Queue Consumer Count` panel shows that jump from 1 to 5, and `Total Queue Messages` drops back toward zero.

**The operational story:**

> The shipping service became the bottleneck. We detected queue growth in Grafana and scaled the consumer deployment to restore throughput.

That is the complete demonstration of why BrokerService and real-time monitoring matter.

---

## Cleanup

```bash
# Remove the /etc/hosts entry added during setup
sudo sed -i '/grafana.brokerservice-monitoring.local/d' /etc/hosts

# Delete the tutorial namespace and everything deployed into it
kubectl delete namespace service-app-project

# Delete the minikube cluster (also removes the Ingress controller)
minikube delete --profile brokerservice-monitoring
```

---

## Troubleshooting

### Metrics not appearing in Grafana

> **Troubleshooting only:** The steps below use `kubectl port-forward` to inspect Prometheus directly. This is a debugging tool, not the normal way to access anything in this tutorial.

Temporarily port-forward Prometheus and check that the broker target is `UP`:

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus \
  -n service-app-project 9090:9090 > /tmp/prometheus-pf.log 2>&1 &
```

Open http://localhost:9090/targets and find `messaging-service-monitor`. The error shown on a failing target will identify whether the problem is TLS, DNS, or authentication.

Also verify the `prometheus-cert` secret exists — the Operator requires it to authorise Prometheus:

```bash
kubectl get secret prometheus-cert -n service-app-project
kubectl get servicemonitor messaging-service-monitor -n service-app-project -o yaml | grep "release: prometheus"
```

### Panel shows "No data" but the target is UP

Search for `broker_queue` in the Prometheus UI to see what metric names your Operator version exposes. If they differ from what the recording rules expect, update the `expr` fields in the PrometheusRule and the dashboard JSON to match.

### Pipeline not flowing

Check deployments and binding secrets:

```bash
kubectl get deployment -n service-app-project
kubectl get secret -n service-app-project | grep binding-secret
```

Check logs for connection errors:

```bash
kubectl logs -n service-app-project deployment/order-processor-app --tail=30 | grep -i error
```
