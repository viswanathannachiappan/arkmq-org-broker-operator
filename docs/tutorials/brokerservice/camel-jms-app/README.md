# camel-jms-app

A Camel Quarkus JMS application used as the pipeline workload in the
[BrokerService Monitoring tutorial](../brokerservice.md).

The same container image is deployed multiple times with different `APP_ROLE`
environment variables, so a single image drives the entire order-processing
pipeline without rebuilding.

## Roles

| `APP_ROLE` | Consumes | Produces |
|---|---|---|
| `generator` | — | `ORDERS.NEW` |
| `processor` | `ORDERS.NEW` | `ORDERS.PROCESSED` |
| `shipping` | `ORDERS.PROCESSED` | `ORDERS.SHIPPED` |
| `delivery` | `ORDERS.SHIPPED` | `ORDERS.DELIVERED` |
| `sink` | any queue | — (drain) |

All other behaviour (queue names, message rate, processing delay, concurrency)
is controlled by environment variables. See
[`application.properties`](src/main/resources/application.properties) for the
full list.

## Build the container image

The `Containerfile` is a multi-stage build: Maven and the JDK run inside the
builder stage, so the only local prerequisite is a container build tool (Docker
or Podman).

### Using Minikube's built-in Docker daemon (recommended for the tutorial)

```bash
# Point your shell at Minikube's Docker daemon so the image is available
# inside the cluster without a registry push.
eval $(minikube docker-env --profile brokerservice-monitoring)

docker build -f docs/tutorials/brokerservice/camel-jms-app/Containerfile docs/tutorials/brokerservice/camel-jms-app/ -t camel-jms-app:latest
```

### Using a local Docker or Podman daemon

```bash
docker build -f docs/tutorials/brokerservice/camel-jms-app/Containerfile docs/tutorials/brokerservice/camel-jms-app/ -t camel-jms-app:latest
# then load into Minikube
minikube image load camel-jms-app:latest --profile brokerservice-monitoring
```

## Using the image

The tutorial Kubernetes Deployments reference the image as:

```yaml
image: camel-jms-app:latest
imagePullPolicy: Never
```

`imagePullPolicy: Never` tells Kubernetes to use the image already present in
the node's local store rather than contacting a registry.
