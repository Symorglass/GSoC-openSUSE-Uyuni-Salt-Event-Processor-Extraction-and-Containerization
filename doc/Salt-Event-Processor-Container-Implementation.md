# Salt Event Processor Container Implementation

**Goal**: Package the extracted Salt Event Processor as a deployable container

**Context**: After successfully extracting the Salt Event Processor into a standalone service, we need to containerize it for production deployment. The container must include all runtime dependencies, configuration files, and startup scripts while maintaining compatibility with the existing Uyuni ecosystem.

**Approach**: Create a dedicated container image that inherits from a minimal base , installs necessary dependencies (which takes a lot of test), and packages the runtime.

---

## Container Architecture Design

### Container Structure

```
containers/server-salt-event-processor-image/
├── Dockerfile                                   # Container definition
├── root/                                        # Runtime files directory
│   ├── etc/rhn/saltEventProcessor.conf          # Configuration file
│   ├── usr/sbin/mgr_salt_event_processor        # Startup script
│   └── usr/share/rhn/classes/                   # Logging config
│       └── log4j2_saltEventProcessor.xml
├── root.tar.gz                                  # Compressed runtime files
├── server-salt-event-processor-image.changes    # Build changelog
├── _service                                     # Build service config
└── tito.props                                   # Packaging properties

```

**`root.tar.gz`**: Packages all runtime files into a single archive for efficient container building

- **Configuration Files**:
    - `/etc/rhn/saltEventProcessor.conf` - Runtime parameters and JVM settings
    - `/usr/share/rhn/classes/log4j2_saltEventProcessor.xml` - Logging configuration
- **Executable Files**:
    - `/usr/sbin/mgr_salt_event_processor` - Main startup script with proper PATH location

### Image Implementation Plan

1. **Start from minimum base Image:** referred to coco-attestation service
2. **Dependency Layering**: Install only necessary packages to keep image size manageable, add in package one by one and test if container is workable
3. **Configuration Externalization**: Runtime files packaged in a compressed file separately from base dependencies 
4. **Health Monitoring**: Built-in health checks for container orchestration 
5. **Production Ready set up**: Proper labels, versioning, and startup configuration

---

## Dockerfile Implementation

### Complete Dockerfile

```docker
# SPDX-License-Identifier: MIT
#!BuildTag: uyuni/server-salt-event-processor:latest

ARG BASE=registry.suse.com/bci/bci-base:15.6
FROM $BASE

# Install main packages
RUN zypper ref && zypper --non-interactive install \
    java-17-openjdk-headless \
    spacewalk-java \
    spacewalk-taskomatic \
    procps

ADD --chown=root:root root.tar.gz /

# Labels for container metadata
ARG PRODUCT=Uyuni
ARG VENDOR="Uyuni project"
ARG URL="https://www.uyuni-project.org/"
ARG REFERENCE_PREFIX="registry.opensuse.org/uyuni"

# Build Service required labels
LABEL org.opencontainers.image.name=server-salt-event-processor-image
LABEL org.opencontainers.image.title="${PRODUCT} Salt Event Processor container"
LABEL org.opencontainers.image.description="${PRODUCT} Salt Event Processor microservice"
LABEL org.opencontainers.image.created="%BUILDTIME%"
LABEL org.opencontainers.image.vendor="${VENDOR}"
LABEL org.opencontainers.image.url="${URL}"
LABEL org.opencontainers.image.version=5.1.6
LABEL org.openbuildservice.disturl="%DISTURL%"
LABEL org.opensuse.reference="${REFERENCE_PREFIX}/server-salt-event-processor:${PRODUCT_VERSION}.%RELEASE%"

# Health check configuration
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD ["pgrep", "-f", "com.suse.saltevent.SaltEventProcessor"]

# Default command
CMD ["/usr/sbin/mgr_salt_event_processor"]
```

**1. Base Layer**: SUSE BCI 15.6

- Compatible with SUSE/openSUSE ecosystem

**2. Package Installation**

```docker
RUN zypper ref && zypper --non-interactive install \
    java-17-openjdk-headless \
    spacewalk-java \
    spacewalk-taskomatic \
    procps
```

**Packages**:

- **java-17-openjdk-headless**: JVM runtime
- **spacewalk-java**: Contains rhn.jar with all Uyuni Java classes
- **spacewalk-taskomatic**: Additional logging log4j
- **procps**: Process utilities for health checks (pgrep, ps, etc.)

**Installation Strategy**:

- Single RUN command to minimize layers
- `-non-interactive` flag for automated builds
- `zypper ref` to refresh package indexes

**3. Runtime Files Layer**

```docker
ADD --chown=root:root root.tar.gz /
```

Packaged all the runtime files into a `root.tar.gz` , and use ADD to automatically extracts it when image is run.

**4. Metadata Labels Layer**

**5. Health Check Configuration**

```docker
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD ["pgrep", "-f", "com.suse.saltevent.SaltEventProcessor"]
```

**Health Check Strategy**:

- **Interval**: Check every 30 seconds
- **Timeout**: Allow 10 seconds for check to complete
- **Start Period**: 60 seconds grace period for startup
- **Retries**: 3 failed checks before marking unhealthy
- **Method**: Process check using pgrep with class name pattern

**Production Benefits**:

- Container orchestrators can detect failed containers
- Automatic restart of unhealthy containers
- Load balancers can route traffic away from unhealthy instances

---

## Next Step:

- Build image using OBS
- Test image