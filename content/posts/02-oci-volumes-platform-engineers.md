+++
title = "Building Reproducible Agent Distribution with Kubernetes Image Volumes"
date = 2025-05-15
draft = true

[taxonomies]
tags = ["kubernetes", "platform-engineering", "image-volumes", "observability", "automation", "devops", "containerd", "infrastructure"]

[extra]
author = "Dominic Lüchinger"
+++

As platform engineers, we constantly face the challenge of distributing instrumentation agents, configuration files,
and other static resources across our Kubernetes clusters without bloating application images or complicating
deployment pipelines.
Kubernetes 1.33's beta Image Volume feature offers an elegant solution that aligns perfectly with modern container practices.

## The Platform Engineering Challenge

Traditional approaches to agent distribution create several pain points:

- **Image Bloat**: Embedding agents in every application image increases size and attack surface
- **Version Skew**: Different applications may bundle different agent versions
- **Build Complexity**: Teams must remember to include agents in their Dockerfiles
- **Update Friction**: Agent updates require rebuilding and redeploying all applications

## Image Volume: A Game Changer

The Image Volume feature allows mounting container images directly as read-only volumes in Kubernetes pods.
This enables a clean separation of concerns where application images contain only application code,
while agents and other resources are distributed through the container registry as separate OCI artifacts.

## Implementation Deep Dive

My [demo repository](https://github.com/dol/k8s-oci-volume-source-demo) showcases a complete implementation using
OpenTelemetry Java agents. Here's how the platform components work together:

### Custom Infrastructure Setup

```bash
# Build custom Kind with containerd v2.1.1
./01-build-custom-kind-image.sh

# Deploy cluster with ImageVolume feature gate enabled
./02-kind-with-registry.sh
```

The infrastructure setup requires careful attention to containerd versions and feature gates.
Standard Kind images don't support Image Volumes, necessitating a custom build with containerd v2.1.1.
Note that while Image Volumes graduated to beta in Kubernetes v1.33, the feature is still disabled by default
and requires explicit enablement via feature gates.

### Reproducible OCI Artifact Creation

The [artifact creation script](https://github.com/dol/k8s-oci-volume-source-demo/blob/main/03-artifact-javaagent-upload.sh)
demonstrates best practices for creating reproducible OCI images:

```bash
# Create reproducible tar layer
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
```

Key considerations for platform teams:

- **Reproducibility**: Consistent builds regardless of build environment
- **Metadata Preservation**: Agent version and build timestamps maintained
- **Registry Integration**: ORAS tooling for OCI artifact management

### Automated Deployment Patterns

The deployment automation demonstrates how platform teams can standardize agent integration:

```bash
# Deploy application with automatic agent injection
./04-deploy-spring-hello-world.sh

# Add observability dashboard
./04a-deploy-aspire-dashboard.sh
```

## Platform Benefits

### Operational Excellence

- **Centralized Agent Management**: Single source of truth for agent versions
- **Zero-Downtime Updates**: Update agents without application restarts
- **Consistent Tooling**: Standard OCI registry tools for artifact management

### Developer Experience

- **Simplified Dockerfiles**: No agent-specific configuration required
- **Faster Builds**: Smaller application images build faster
- **Reduced Cognitive Load**: Developers focus on application code, not infrastructure concerns

### Security and Compliance

- **Vulnerability Management**: Agent security updates don't require application rebuilds
- **Supply Chain Transparency**: Clear separation between application and tooling artifacts
- **Immutable Infrastructure**: Version-pinned agents ensure consistent behavior

## Production Considerations

### Registry Strategy

- Use enterprise-grade registries for production workloads
- Implement proper RBAC for OCI artifact access
- Consider geo-distributed registries for global deployments

### Monitoring and Alerting

- Track Image Volume mount success/failure rates using new kubelet metrics
- Monitor agent version distribution across clusters
- Alert on registry connectivity issues
- Utilize new kubelet metrics: `kubelet_image_volume_requested_total`,
  `kubelet_image_volume_mounted_succeed_total`, and `kubelet_image_volume_mounted_errors_total`

### Version Management

- Implement semantic versioning for agent artifacts
- Use admission controllers to enforce agent version policies
- Maintain compatibility matrices between agents and applications

## Automation Integration

The demo includes complete CI/CD patterns that platform teams can adapt:

1. **Infrastructure as Code**: Kind cluster configuration with proper feature gates
2. **Artifact Pipelines**: Automated OCI image creation and publishing
3. **Deployment Automation**: Kustomize-based application deployment
4. **Cleanup Procedures**: Complete environment teardown for testing

## Looking Forward

Image Volumes graduated to beta in Kubernetes v1.33, representing a significant step toward cleaner container architectures.
However, the feature remains disabled by default due to varying container runtime support. Platform teams should:

1. **Experiment Now**: Build proof-of-concepts with non-critical workloads
2. **Plan Migration**: Identify applications that would benefit from agent externalization
3. **Prepare Tooling**: Adapt CI/CD pipelines for OCI artifact workflows
4. **Train Teams**: Educate developers on the new deployment patterns

The [complete demo](https://github.com/dol/k8s-oci-volume-source-demo) provides a foundation for platform teams to
explore this technology.
With Image Volumes now in beta status, it's an excellent time to begin incorporating this capability into your platform strategy.

---

*The complete working demo is available at [https://github.com/dol/k8s-oci-volume-source-demo](https://github.com/dol/k8s-oci-volume-source-demo),
including all scripts, manifests, and documentation needed to reproduce this setup.*
