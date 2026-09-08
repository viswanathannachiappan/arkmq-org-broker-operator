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

### Build the Camel Pipeline Image

The Camel pipeline image is built directly into Minikube's image store so no
registry push is needed. The source lives alongside this tutorial in
[`camel-jms-app/`](camel-jms-app/). The `Containerfile` is a multi-stage build
— Maven and the JDK run inside the builder container, so no local JDK or Maven
installation is required.

```bash {"stage":"init", "label":"build camel jms image", "rootdir":"$initial_dir", "runtime":"bash"}
eval $(minikube docker-env --profile brokerservice-monitoring)
docker build -f tutorials/brokerservice/camel-jms-app/Containerfile tutorials/brokerservice/camel-jms-app/ -t camel-jms-app:latest
```

The first build takes a few minutes while Maven downloads dependencies and
compiles the Quarkus application. Subsequent builds reuse the cached dependency
layer and are much faster.

### Create Namespace

```bash {"stage":"init", "runtime":"bash"}
kubectl create namespace service-app-project
kubectl config set-context --current --namespace=service-app-project
```

### Install Cert-Manager

```bash {"stage":"init", "label":"install cert-manager", "runtime":"bash"}
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.5/cert-manager.yaml
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
      "version": 27,
      "panels": [

        {
          "id": 100,
          "title": "Memory Overview",
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
          "id": 2,
          "title": "Container Memory %",
          "description": "Current messaging-service container working-set memory as a percentage of its configured Kubernetes memory limit.",
          "type": "stat",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 4,
            "w": 6,
            "x": 0,
            "y": 1
          },
          "targets": [
            {
              "expr": "100 * sum(container_memory_working_set_bytes{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",container!=\"POD\",container!=\"\"}) / clamp_min(sum(kube_pod_container_resource_limits{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",resource=\"memory\",unit=\"byte\"}),1)",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "percent",
              "min": 0,
              "max": 100,
              "color": {
                "mode": "thresholds"
              },
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "yellow",
                    "value": 70
                  },
                  {
                    "color": "red",
                    "value": 85
                  }
                ]
              }
            }
          },
          "options": {
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ]
            },
            "colorMode": "background"
          }
        },

        {
          "id": 3,
          "title": "JVM Heap %",
          "description": "Current JVM heap used as a percentage of the maximum configured JVM heap.",
          "type": "stat",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 4,
            "w": 6,
            "x": 6,
            "y": 1
          },
          "targets": [
            {
              "expr": "100 * sum(jvm_memory_used_bytes{job=\"messaging-service-metrics\",area=\"heap\"}) / clamp_min(sum(jvm_memory_max_bytes{job=\"messaging-service-metrics\",area=\"heap\"}),1)",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "percent",
              "min": 0,
              "max": 100,
              "color": {
                "mode": "thresholds"
              },
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "yellow",
                    "value": 70
                  },
                  {
                    "color": "red",
                    "value": 85
                  }
                ]
              }
            }
          },
          "options": {
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ]
            },
            "colorMode": "background"
          }
        },

        {
          "id": 4,
          "title": "Queue Persistent Data",
          "description": "Total persistent message data currently retained across all Artemis queues.",
          "type": "stat",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 4,
            "w": 6,
            "x": 12,
            "y": 1
          },
          "targets": [
            {
              "expr": "sum(broker_queue_persistent_size{job=\"messaging-service-metrics\"})",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "bytes",
              "min": 0,
              "color": {
                "mode": "fixed",
                "fixedColor": "green"
              }
            }
          },
          "options": {
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ]
            },
            "colorMode": "value"
          }
        },

        {
          "id": 5,
          "title": "Container Memory Headroom",
          "description": "Remaining difference between the messaging-service container memory limit and current working-set memory.",
          "type": "stat",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 4,
            "w": 6,
            "x": 18,
            "y": 1
          },
          "targets": [
            {
              "expr": "sum(kube_pod_container_resource_limits{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",resource=\"memory\",unit=\"byte\"}) - sum(container_memory_working_set_bytes{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",container!=\"POD\",container!=\"\"})",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "bytes",
              "min": 0,
              "color": {
                "mode": "thresholds"
              },
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "red",
                    "value": null
                  },
                  {
                    "color": "yellow",
                    "value": 104857600
                  },
                  {
                    "color": "green",
                    "value": 314572800
                  }
                ]
              }
            }
          },
          "options": {
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ]
            },
            "colorMode": "background"
          }
        },

        {
          "id": 200,
          "title": "Container, JVM & Queue Memory",
          "type": "row",
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 5
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
            "h": 12,
            "w": 24,
            "x": 0,
            "y": 6
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
              "expr": "sum(kube_pod_container_resource_limits{namespace=\"service-app-project\",pod=~\"messaging-service-ss-.*\",resource=\"memory\",unit=\"byte\"})",
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
          "title": "Queue Storage Contribution",
          "type": "row",
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 18
          }
        },

        {
          "id": 31,
          "title": "Queue Storage Contribution (Bar Chart)",
          "description": "Persistent storage consumed by each individual Artemis queue.",
          "type": "barchart",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 8,
            "w": 24,
            "x": 0,
            "y": 19
          },
          "targets": [
            {
              "expr": "sum by (queue)(broker_queue_persistent_size{job=\"messaging-service-metrics\"})",
              "legendFormat": "{{queue}}",
              "refId": "A",
              "instant": true
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "bytes",
              "min": 0,
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "fillOpacity": 85,
                "lineWidth": 1
              }
            }
          },
          "options": {
            "orientation": "horizontal",
            "showValue": "always",
            "groupWidth": 0.7,
            "barWidth": 0.7,
            "legend": {
              "displayMode": "list",
              "placement": "right"
            }
          }
        },

        {
          "id": 32,
          "title": "Current Queue Breakdown",
          "description": "Current persistent bytes, message count, and percentage of total storage for every Artemis queue.",
          "type": "table",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 8,
            "w": 24,
            "x": 0,
            "y": 27
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
            },
            {
              "expr": "100 * sum by (queue)(broker_queue_persistent_size{job=\"messaging-service-metrics\"}) / clamp_min(sum(broker_queue_persistent_size{job=\"messaging-service-metrics\"}), 1)",
              "format": "table",
              "instant": true,
              "refId": "C"
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
                  "Time 2": true,
                  "Time 3": true
                },
                "indexByName": {
                  "queue": 0,
                  "Value #A": 1,
                  "Value #B": 2,
                  "Value #C": 3
                },
                "renameByName": {
                  "queue": "Queue",
                  "Value #A": "Persistent Data",
                  "Value #B": "Messages",
                  "Value #C": "% of Total"
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
                    "value": 240
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
                  }
                ]
              },
              {
                "matcher": {
                  "id": "byName",
                  "options": "% of Total"
                },
                "properties": [
                  {
                    "id": "unit",
                    "value": "percent"
                  },
                  {
                    "id": "min",
                    "value": 0
                  },
                  {
                    "id": "max",
                    "value": 100
                  },
                  {
                    "id": "custom.cellOptions",
                    "value": {
                      "type": "gauge",
                      "mode": "gradient"
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
        },

        {
          "id": 400,
          "title": "JVM Diagnostics",
          "type": "row",
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 35
          }
        },

        {
          "id": 41,
          "title": "JVM GC Activity",
          "description": "JVM garbage collection execution rates.",
          "type": "timeseries",
          "datasource": {
            "type": "prometheus",
            "uid": "prometheus"
          },
          "gridPos": {
            "h": 8,
            "w": 24,
            "x": 0,
            "y": 36
          },
          "targets": [
            {
              "expr": "rate(jvm_gc_collection_seconds_count{job=\"messaging-service-metrics\",gc=~\".*Young.*\"}[5m])",
              "legendFormat": "Young GC rate",
              "refId": "A"
            },
            {
              "expr": "rate(jvm_gc_collection_seconds_count{job=\"messaging-service-metrics\",gc=~\".*Old.*\"}[5m])",
              "legendFormat": "Old GC rate",
              "refId": "B"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "ops",
              "min": 0,
              "custom": {
                "drawStyle": "line",
                "lineWidth": 2,
                "fillOpacity": 10,
                "showPoints": "never",
                "spanNulls": true
              }
            }
          },
          "options": {
            "legend": {
              "displayMode": "table",
              "placement": "bottom",
              "calcs": [
                "lastNotNull",
                "max"
              ]
            }
          }
        }
      ]
    }
EOF
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

Add the Minikube IP to your `/etc/hosts` so the hostname resolves locally:

```bash {"stage":"grafana", "label":"configure hosts", "runtime":"bash"}
export CLUSTER_IP=$(minikube ip --profile brokerservice-monitoring)
echo "${CLUSTER_IP} grafana.brokerservice-monitoring.local" | sudo tee -a /etc/hosts
echo "Grafana available at http://grafana.brokerservice-monitoring.local"
```

```bash {"stage":"grafana", "label":"get grafana password", "runtime":"bash"}
kubectl get secret prometheus-grafana -n service-app-project -o jsonpath='{.data.admin-password}' | base64 -d && echo
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
> The broker is configured with 1 GiB memory limit for this tutorial workload.


```bash {"stage":"deploy_service", "label":"wait for brokerservice", "runtime":"bash"}
kubectl wait BrokerService messaging-service -n service-app-project --for=condition=Ready --timeout=300s
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
      app: "order-processing-pipeline"
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

```bash {"stage":"deploy_app", "label":"wait for order-processor-app cert", "runtime":"bash"}
kubectl wait certificate order-processor-app-cert -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_app", "label":"wait for order-processor-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp order-processor-app -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_app", "label":"wait for shipping-service-app cert", "runtime":"bash"}
kubectl wait certificate shipping-service-app-cert -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_app", "label":"wait for shipping-service-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp shipping-service-app -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_app", "label":"wait for delivery-service-app cert", "runtime":"bash"}
kubectl wait certificate delivery-service-app-cert -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_app", "label":"wait for delivery-service-app brokerapp", "runtime":"bash"}
kubectl wait BrokerApp delivery-service-app -n service-app-project --for=condition=Ready --timeout=300s
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

Then confirm the secrets exist:

```bash {"stage":"deploy_app", "label":"verify binding secrets", "runtime":"bash"}
kubectl get secret -n service-app-project | grep binding-secret
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

> **Note:** The `cert-pemcfg-*` Secrets used by the individual Deployments contain this same configuration. The certificate itself is not stored in these Secrets — each application's certificate comes from its own cert-manager-generated Secret.

### order-generator

`order-generator` produces 5 order messages per second into `ORDERS.NEW`. The rate is controlled by the `MESSAGE_RATE` environment variable in the Camel Deployment. The BrokerApp only declares the messaging capability (`producerOf: ORDERS.NEW`).

To change the message rate, update the Deployment's `MESSAGE_RATE` value — no image rebuild is required.

Before deploying the application, wait for the binding Secret created for its BrokerApp:

```bash {"stage":"deploy_camel", "label":"wait for order-generator binding secret", "runtime":"bash"}
kubectl wait secret order-generator-binding-secret -n service-app-project --for=create --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for order-generator", "runtime":"bash"}
kubectl wait deployment order-generator -n service-app-project --for=condition=Available --timeout=300s
```

### order-processor-app (Order Processor)

```bash {"stage":"deploy_camel", "label":"wait for order-processor-app binding secret", "runtime":"bash"}
kubectl wait secret order-processor-app-binding-secret -n service-app-project --for=create --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for order-processor-app", "runtime":"bash"}
kubectl wait deployment order-processor-app -n service-app-project --for=condition=Available --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for shipping-service-app binding secret", "runtime":"bash"}
kubectl wait secret shipping-service-app-binding-secret -n service-app-project --for=create --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for shipping-service-app", "runtime":"bash"}
kubectl wait deployment shipping-service-app -n service-app-project --for=condition=Available --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for delivery-service-app binding secret", "runtime":"bash"}
kubectl wait secret delivery-service-app-binding-secret -n service-app-project --for=create --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for delivery-service-app", "runtime":"bash"}
kubectl wait deployment delivery-service-app -n service-app-project --for=condition=Available --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for master-sink cert", "runtime":"bash"}
kubectl wait certificate master-sink-app-cert -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for master-sink brokerapp", "runtime":"bash"}
kubectl wait BrokerApp master-sink-app -n service-app-project --for=condition=Ready --timeout=300s
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

```bash {"stage":"deploy_camel", "label":"wait for master-sink binding secret", "runtime":"bash"}
kubectl wait secret master-sink-app-binding-secret -n service-app-project --for=create --timeout=300s
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

```bash {"stage":"verify", "label":"check processor logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/order-processor-app --tail=20
```

```bash {"stage":"verify", "label":"check shipping logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/shipping-service-app --tail=20
```

```bash {"stage":"verify", "label":"check delivery logs", "runtime":"bash"}
kubectl logs -n service-app-project deployment/delivery-service-app --tail=20
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
export BROKER_POD=$(kubectl get pods \
  -n service-app-project \
  -l ActiveMQArtemis=messaging-service \
  -o jsonpath='{.items[0].metadata.name}')
export BROKER_FQDN="${BROKER_POD}.messaging-service-hdls-svc.service-app-project.svc.cluster.local"
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

Scaling gives you five consumers, but they still have the 2-second processing delay — so aggregate throughput is still only ~2.5 msg/s at this point. The backlog may therefore continue growing during the rollout. The next step restores the 25 ms delay; after that rollout completes, aggregate theoretical capacity rises to ~200 msg/s — though actual throughput will be lower due to JMS and scheduling overhead. With the generator still producing 5 msg/s, the backlog can drain at up to roughly 195 msg/s under idealized conditions, so even a large accumulated backlog clears quickly:

```bash {"stage":"scenario_scale", "label":"reset shipping delay", "runtime":"bash"}
kubectl set env deployment/shipping-service-app \
  PROCESSING_DELAY_MS=25 \
  -n service-app-project
kubectl rollout status deployment/shipping-service-app -n service-app-project --timeout=120s
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
