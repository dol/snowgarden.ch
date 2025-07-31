+++
title = "Auto-Instrumenting Java Apps with Kubernetes Image Volumes"
date = 2025-05-15
draft = true

[taxonomies]
tags=["kubernetes", "java", "opentelemetry", "observability", "oci", "containers", "platform-engineering", "devops", "kind", "spring-boot"]

[extra]
repo_view = true
repo_url = "https://github.com/dol/k8s-oci-volume-source-demo"
+++

# Auto-Instrumenting Java Apps with Kubernetes Image Volumes

The [Kubernetes v1.33 release](https://kubernetes.io/blog/2025/04/29/kubernetes-v1-33-image-volume-beta/) marks an important milestone with the **Image Volumes** feature graduating to beta status. This feature allows you to mount container images directly as read-only volumes in your Kubernetes pods, creating exciting new possibilities for software delivery patterns.

In this blog post, I'll walk through a practical implementation of this feature using [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) to demonstrate how to auto-instrument a Java application with [OpenTelemetry](https://opentelemetry.io/) without modifying the application container itself.

> **📁 Complete Code Available**: All the code and configuration files for this tutorial are available in the [k8s-oci-volume-source-demo](https://github.com/dol/k8s-oci-volume-source-demo) GitHub repository. You can clone it and follow along or use it as a reference.

## What are Image Volumes?

Image Volumes were introduced as an alpha feature in Kubernetes v1.31 as part of [KEP-4639](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4639-oci-volume-source). With v1.33, they have graduated to beta, adding support for `subPath` and `subPathExpr` mounts, along with new metrics for tracking image volume usage.

This feature allows you to reference container images as volumes in Kubernetes pods, giving you direct access to the container image's filesystem. The volumes are mounted read-only, maintaining security and immutability.

> **Note:** The feature is still disabled by default, as not all container runtimes fully support it yet. [Containerd v2.1.0](https://github.com/containerd/containerd/releases/tag/v2.1.0) supports this feature, which is what we'll be using in this tutorial.

## Why Image Volumes Matter

This feature can revolutionize how we deliver and manage certain types of content in Kubernetes:

1. **Separate content from application containers** - Keep your application containers lean while mounting heavy dependencies separately
2. **Agent distribution** - Distribute monitoring agents without modifying application images
3. **Simplified versioning** - Update content independently from application code
4. **Reduced deployment complexity** - Avoid custom init containers and sidecars

In our example, we'll use Image Volumes to mount an OpenTelemetry Java agent into a Spring Boot application without embedding it in the application container.

## Prerequisites

To follow along with this tutorial, you'll need:

- [Docker](https://www.docker.com/get-started/) installed on your machine
- [`git`](https://git-scm.com/downloads) for cloning repositories
- [`curl`](https://curl.se/download.html) for downloading files
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl) for interacting with Kubernetes
- Basic understanding of Kubernetes concepts

## Getting Started

Clone the demo repository to get started:

```bash
git clone https://github.com/dol/k8s-oci-volume-source-demo.git
cd k8s-oci-volume-source-demo
```

The repository contains everything needed to run the complete demo, including:

- **Setup scripts**: [`01-build-custom-kind-image.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/01-build-custom-kind-image.sh), [`02-kind-with-registry.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/02-kind-with-registry.sh)
- **OCI artifact creation**: [`03-artifact-javaagent-upload.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/03-artifact-javaagent-upload.sh)
- **Application deployments**: [`04-deploy-spring-hello-world.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/04-deploy-spring-hello-world.sh), [`04a-deploy-aspire-dashboard.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/04a-deploy-aspire-dashboard.sh)
- **Kubernetes manifests**: [`spring-hello-world/`](https://github.com/dol/k8s-oci-volume-source-demo/tree/main/spring-hello-world), [`aspire-dashboard/`](https://github.com/dol/k8s-oci-volume-source-demo/tree/main/aspire-dashboard)
- **Cleanup script**: [`05-cleanup.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/05-cleanup.sh)

## Step 1: Building a Custom Kind Cluster with Image Volumes Support

First, we need to build a custom Kind node image with a sufficiently recent version of containerd (v2.1.0) that supports the Image Volumes feature.

The complete script is available as [`01-build-custom-kind-image.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/01-build-custom-kind-image.sh) in the repository:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Working directory
WORKDIR=$(pwd)
KIND_REPO="kind"
K8S_TAR="kubernetes-server-linux-amd64.tar.gz"
K8S_VERSION="v1.33.0"
CONTAINERD_VERSION="v2.1.0"
TAG="oci-source-demo"

echo "Building custom kind base image with containerd ${CONTAINERD_VERSION} and Kubernetes ${K8S_VERSION}..."

# Step 1: Verify if kind repo is already cloned, if not clone it
if [ ! -d "${KIND_REPO}" ]; then
  echo "Cloning kind repository..."
  git clone https://github.com/kubernetes-sigs/kind.git
  cd "${KIND_REPO}"
  git checkout v0.27.0
else
  echo "Kind repository already exists, checking out v0.27.0..."
  cd "${KIND_REPO}"
  git fetch --all --tags
  git checkout v0.27.0
fi

# Step 2: Build the base image with custom containerd version
echo "Building custom base image with containerd ${CONTAINERD_VERSION}..."
cd images/base
make quick EXTRA_BUILD_OPT="--build-arg CONTAINERD_VERSION=${CONTAINERD_VERSION}" TAG=${TAG}

# Return to the working directory
cd "${WORKDIR}"

# Step 3: Check if kubernetes tarball exists, if not download it
if [ ! -f "${K8S_TAR}" ]; then
  echo "Downloading Kubernetes server tarball..."
  curl -L "https://dl.k8s.io/${K8S_VERSION}/${K8S_TAR}" -o "${K8S_TAR}"
else
  echo "Kubernetes server tarball already exists"
fi

# Step 4: Build the node image using the custom base image
echo "Building node image using custom base image..."
kind build node-image ./${K8S_TAR} --type file --base-image gcr.io/k8s-staging-kind/base:${TAG}
```

This script performs the following operations:

1. Clones the [Kind repository](https://github.com/kubernetes-sigs/kind) and checks out version v0.27.0
2. Builds a custom base image with [containerd v2.1.0](https://github.com/containerd/containerd/releases/tag/v2.1.0)
3. Downloads the [Kubernetes v1.33.0](https://kubernetes.io/releases/) server tarball if needed
4. Builds a Kind node image using the custom base image and Kubernetes binaries

## Step 2: Creating a Kind Cluster with a Local Registry

Next, we'll create a Kind cluster with our custom image and set up a local registry for our OCI artifacts using the [`02-kind-with-registry.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/02-kind-with-registry.sh) script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Create registry container unless it already exists
reg_name='kind-registry'
reg_port='5001'
if [ "$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)" != 'true' ]; then
  docker run \
    -d --restart=always \
    -p "127.0.0.1:${reg_port}:5000" \
    --network bridge \
    --name "${reg_name}" \
    registry:3

  # Wait for the registry to be ready
  while ! curl -s "http://localhost:${reg_port}/v2/" >/dev/null; do
    echo "Waiting for registry to be ready..."
    sleep 1
  done
  echo "Registry is ready"
else
  echo "Registry already exists"
fi

# 2. Create kind cluster with containerd registry config dir enabled
cat <<EOF | kind create cluster --image kindest/node:latest --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
featureGates:
  "ImageVolume": true
EOF

# 3. Add the registry config to the nodes
REGISTRY_DIR="/etc/containerd/certs.d/localhost:${reg_port}"
for node in $(kind get nodes); do
  docker exec "${node}" mkdir -p "${REGISTRY_DIR}"
  cat <<EOF | docker exec -i "${node}" cp /dev/stdin "${REGISTRY_DIR}/hosts.toml"
[host."http://${reg_name}:5000"]
EOF
done

# 4. Connect the registry to the cluster network if not already connected
if [ "$(docker inspect -f='{{json .NetworkSettings.Networks.kind}}' "${reg_name}")" = 'null' ]; then
  docker network connect "kind" "${reg_name}"
fi

# 5. Document the local registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${reg_port}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF
```

This script:

1. Creates a local [Docker registry](https://docs.docker.com/registry/) container
2. Creates a Kind cluster with the custom node image and enables the `ImageVolume` feature gate
3. Configures containerd in each node to use the local registry
4. Connects the registry to the Kind network
5. Creates a ConfigMap to document the local registry

## Step 3: Creating an OCI Artifact for the OpenTelemetry Java Agent

Now we'll package the [OpenTelemetry Java agent](https://github.com/open-telemetry/opentelemetry-java-instrumentation) as an [OCI artifact](https://github.com/opencontainers/artifacts) and push it to our local registry using the [`03-artifact-javaagent-upload.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/03-artifact-javaagent-upload.sh) script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Create a temporary directory for the OCI image
TMP_DIR="$(mktemp -d)"

# Constants
GITHUB_REPO="open-telemetry/opentelemetry-java-instrumentation"
AGENT_VERSION="v2.15.0"
AGENT_URL="https://github.com/${GITHUB_REPO}/releases/download/${AGENT_VERSION}/opentelemetry-javaagent.jar"
AGENT_JAR_FILE="opentelemetry-javaagent.jar"
AGENT_DIR="${TMP_DIR}/opentelemetry-javaagent"
AGENT_FILE="${AGENT_DIR}/${AGENT_JAR_FILE}"
AGENT_FILE_DOWNLOAD_HEADERS="${AGENT_DIR}/${AGENT_JAR_FILE}.curl.headers"
REGISTRY="localhost:5001"
ARTIFACT_NAME="opentelemetry-javaagent"

echo "Step 1: Downloading OpenTelemetry Java Agent..."
# Create agent directory if it doesn't exist
mkdir -p "${AGENT_DIR}"

curl -L \
  --dump-header "${AGENT_FILE_DOWNLOAD_HEADERS}" \
  --output "${AGENT_FILE}" \
  "${AGENT_URL}"

# Extract the Last-Modified date from headers
AGENT_PUBLISH_DATE="$(grep -i '^last-modified:' "${AGENT_FILE_DOWNLOAD_HEADERS}" | sed -E 's/^Last-Modified:[[:space:]]*//I')"

# Set the file's modification time to match the server's Last-Modified time
if TOUCH_DATE=$(date -d "${AGENT_PUBLISH_DATE}" "+%Y%m%d%H%M.%S" 2>/dev/null) && [ -n "$TOUCH_DATE" ]; then
  touch -t "${TOUCH_DATE}" "${AGENT_FILE}"
  echo "Set file timestamp to match server's Last-Modified: ${AGENT_PUBLISH_DATE}"
else
  echo "Warning: Could not convert date format for touch command"
fi

echo "Step 2: Creating OCI image from Dockerfile..."

TAR_LAYER_PATH="${TMP_DIR}/layer.tar"
# Create a tar layer from the agent directory and make it reproducible
tar c -f "${TAR_LAYER_PATH}" \
  -C "${AGENT_DIR}" \
  --sort=name \
  --format=posix \
  --pax-option=exthdr.name=%d/PaxHeaders/%f \
  --pax-option=delete=atime,delete=ctime \
  --owner=0 \
  --group=0 \
  --numeric-owner \
  --mode=0444 \
  --clamp-mtime --mtime=0 \
  "${AGENT_JAR_FILE}"

# Calculate layer diff
TAR_LAYER_DIFF="$(sha256sum "${TAR_LAYER_PATH}" | head -c 64)"

# Compress layer and make it reproducible
gzip --best --no-name "${TAR_LAYER_PATH}"

# Create config
CONFIG_PATH="${TMP_DIR}/config.json"
printf '{"architecture":"amd64","os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:%s"]}}' "${TAR_LAYER_DIFF}" > "${CONFIG_PATH}"

# Create layout
LAYOUT_REF="${TMP_DIR}/layout:latest"

# Create OCI image with annotations
IMAGE_CREATED_DATE=$(date -d "${AGENT_PUBLISH_DATE}" --rfc-3339=seconds 2>/dev/null | sed 's/ /T/' || date --rfc-3339=seconds | sed 's/ /T/')

(cd "${TMP_DIR}"; oras push --disable-path-validation \
  --config "${CONFIG_PATH}:application/vnd.oci.image.config.v1+json" \
  --oci-layout "${LAYOUT_REF}" \
  --annotation "org.opencontainers.image.created=${IMAGE_CREATED_DATE}" \
  "layer.tar.gz:application/vnd.oci.image.layer.v1.tar+gzip")

# Push image to local registry
echo "Step 3: Uploading OCI image to registry..."
oras cp --from-oci-layout "${LAYOUT_REF}" "${REGISTRY}/${ARTIFACT_NAME}:${AGENT_VERSION}"

echo "Successfully uploaded ${AGENT_FILE} to ${REGISTRY}/${ARTIFACT_NAME}:${AGENT_VERSION}"

# Clean up the temporary directory
rm -rf "${TMP_DIR}"
```

This script:

1. Downloads the OpenTelemetry Java agent JAR file from the [official GitHub releases](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases)
2. Creates a reproducible OCI image layer containing the agent using [ORAS](https://oras.land/) (OCI Registry As Storage)
3. Configures the OCI image with appropriate metadata
4. Pushes the image to our local registry

## Step 4: Deploying a Java Application with Image Volume Mount

Now we'll deploy a [Spring Boot](https://spring.io/projects/spring-boot) application that mounts the OpenTelemetry agent from the OCI image using the [`04-deploy-spring-hello-world.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/04-deploy-spring-hello-world.sh) script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Constants
NAMESPACE="spring-hello-world"

echo "Deploying Spring Hello World application..."

# Create namespace if it doesn't exist, or delete and recreate it
if kubectl get namespace "${NAMESPACE}" > /dev/null 2>&1; then
  echo "Deleting existing namespace ${NAMESPACE}..."
  kubectl delete namespace "${NAMESPACE}" --wait=true
fi

echo "Creating namespace ${NAMESPACE}..."
kubectl create namespace "${NAMESPACE}"

echo "Applying Kubernetes resources with kustomize..."
kubectl apply -k "spring-hello-world"

echo "Waiting for Spring Hello World deployment to be ready..."
kubectl wait --for=condition=ready --timeout=120s pod -l app=spring-hello-world -n ${NAMESPACE}

# Get the NodePort for Spring Hello World
SPRING_NODE_PORT=$(kubectl get svc spring-hello-world-service -n ${NAMESPACE} -o=jsonpath='{.spec.ports[0].nodePort}')

# Get the node IP from the Spring Hello World pod
NODE_IP=$(kubectl get pod -l app=spring-hello-world -n ${NAMESPACE} -o jsonpath='{.items[0].status.hostIP}')

echo
echo "Deployment completed successfully!"
echo "----------------------------------------"
echo "Spring Hello World application is accessible at: http://${NODE_IP}:${SPRING_NODE_PORT}"
```

The deployment YAML for this application includes the critical Image Volume configuration. You can find the complete Kubernetes manifests in the [`spring-hello-world`](https://github.com/dol/k8s-oci-volume-source-demo/tree/main/spring-hello-world) directory:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-hello-world
  namespace: spring-hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-hello-world
  template:
    metadata:
      labels:
        app: spring-hello-world
    spec:
      containers:
        - name: spring-hello-world
          image: springio/hello-world:0.0.1-SNAPSHOT
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: JAVA_TOOL_OPTIONS
              value: -javaagent:/mnt/javaagent/opentelemetry-javaagent.jar
            - name: OTEL_SERVICE_NAME
              value: spring-hello-world
            - name: OTEL_METRICS_EXPORTER
              value: otlp
            - name: OTEL_LOGS_EXPORTER
              value: otlp
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://aspire-dashboard-service.aspire-dashboard:4317
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          volumeMounts:
            - name: otel-agent
              mountPath: /mnt/javaagent
              readOnly: true
      volumes:
        - name: otel-agent
          image:
            reference: localhost:5001/opentelemetry-javaagent:v2.15.0
```

Note the key sections:

- `volumes` section defines an image volume referencing our OpenTelemetry agent OCI image
- `volumeMounts` mounts this image volume at `/mnt/javaagent` in the container
- `JAVA_TOOL_OPTIONS` environment variable configures Java to use the agent from the mounted path - see the [JVM documentation](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/envvars002.html) for more details

## Step 5: Deploying the Aspire Dashboard for Observability

Finally, we'll deploy the [.NET Aspire Dashboard](https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/dashboard) to visualize the telemetry data collected by the OpenTelemetry agent using the [`04a-deploy-aspire-dashboard.sh`](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/04a-deploy-aspire-dashboard.sh) script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Constants
NAMESPACE="aspire-dashboard"

echo "Deploying Aspire Dashboard..."

# Create namespace if it doesn't exist, or delete and recreate it
if kubectl get namespace "${NAMESPACE}" > /dev/null 2>&1; then
  echo "Deleting existing namespace ${NAMESPACE}..."
  kubectl delete namespace "${NAMESPACE}" --wait=true
fi

echo "Creating namespace ${NAMESPACE}..."
kubectl create namespace "${NAMESPACE}"

echo "Applying Kubernetes resources with kustomize..."
kubectl apply -k "aspire-dashboard"

echo "Waiting for Aspire Dashboard deployment to be ready..."
kubectl wait --for=condition=ready --timeout=120s pod -l app=aspire-dashboard -n ${NAMESPACE}

# Get the NodePort for Aspire Dashboard
ASPIRE_NODE_PORT=$(kubectl get svc aspire-dashboard-service -n ${NAMESPACE} -o=jsonpath='{.spec.ports[0].nodePort}')

# Get the node IP from the Aspire Dashboard pod
NODE_IP=$(kubectl get pod -l app=aspire-dashboard -n ${NAMESPACE} -o jsonpath='{.items[0].status.hostIP}')

echo
echo "Deployment completed successfully!"
echo "----------------------------------------"
echo "Aspire Dashboard is accessible at: http://${NODE_IP}:${ASPIRE_NODE_PORT}"
```

## Benefits of Using Image Volumes for Auto-Instrumentation

This approach offers several significant advantages:

1. **Separation of Concerns**: The instrumentation agent is completely decoupled from the application container
2. **Immutability**: The agent is delivered as an immutable OCI artifact
3. **Versioning**: The agent can be updated independently of the application
4. **Consistency**: All applications can use the same agent without duplicating it in each image
5. **Zero Application Changes**: No need to modify application Dockerfiles or rebuild images

## Conclusion

Kubernetes v1.33's Image Volumes beta feature provides a powerful new way to manage content delivery to containers. By leveraging this feature for auto-instrumentation, we've demonstrated a clean, efficient approach to adding observability to applications without modifying their container images.

This pattern can be extended to other use cases such as:

- Mounting configuration files or scripts
- Adding shared libraries or dependencies
- Distributing ML models to inference containers
- Sharing static assets across multiple applications

As container runtimes continue to improve their support for this feature, we can expect to see widespread adoption of these patterns in production Kubernetes environments.

## Further Reading

- **[Complete Demo Repository](https://github.com/dol/k8s-oci-volume-source-demo)** — All the code and configuration files used in this tutorial
- [Kubernetes v1.33 Image Volumes Beta Announcement](https://kubernetes.io/blog/2025/04/29/kubernetes-v1-33-image-volume-beta/)
- [Using OCI Volume Source in Kubernetes Pods](https://sestegra.medium.com/using-oci-volume-source-in-kubernetes-pods-06d62fb72086)
- [OpenTelemetry Java Instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
- [Kind: Kubernetes in Docker](https://kind.sigs.k8s.io/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Containerd Documentation](https://containerd.io/docs/)
- [ORAS CLI Documentation](https://oras.land/docs/)
