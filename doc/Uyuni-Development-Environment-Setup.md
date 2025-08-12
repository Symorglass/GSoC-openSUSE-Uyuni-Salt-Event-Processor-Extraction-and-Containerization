# Uyuni Development Environment Setup

**Phase**: Environment setup

**Goal**: Setting up containerized development environment for Salt Event Processor extraction

**Context**: Need to work on extracting Salt event processor from Uyuni into separate container. Started with very basic knowledge of Uyuni deployment process.

**Main confusion points to address**:

- We have two different “containers” - Uyuni server and development environment
- Podman, Docker ecosystem
- User permissions (root vs regular user for different tasks)
- Build/deploy process with Ant

---

## Problem Investigation

### Container Architecture Understanding

**Found**: Uyuni uses two distinct container concepts:

1. **Production Uyuni Server**(via`mgradm install podman`) - the actual running system
2. **DevContainer**- development environment with tools/dependencies

These are separate containers serving different purposes.

### General Deployment Flow

In Source Code (~/uyuni/java) 
  1. Ant Build (manager-build.xml)
  2. Deploy to Container (podman)
  3. Test in Running Uyuni 


**Basic Build Command**:

```bash
ant -f manager-build.xml -Ddeploy.mode=container refresh-branding-jar deploy-restart
```

---

## My Deployment Flow

### 1. Uyuni Server Installation and Deployment

This provide the basic production environment for uyuni development

[Uyuni Server Production environment set up ](Uyuni-server-production-environment-setup.md)

### 2. Development Workflow Established

```bash
# Edit code then build uyuni/java as root 
sudo ant -f manager-build.xml refresh-branding-jar -Ddeploy.mode=container deploy-restart

# Test in container shell
mgrctl exec -it bash
```

---

## Key Takeaways

Knowledge Gaps Identified during development

- Java/Ant build system mechanics
- Container networking for development vs production

**Techical takeaways:**

- **Container isolation is per-user in Podman** - root and regular user have completely separate container storage
- **`mgrctl`** is the bridge between host server and Uyuni server container management, uyuni-tools use mgrctl to manage containers
- **Ant build system** handles the complexity of deploying into running containers
- **DevContainers** in Uyuni provide development tooling, not runtime environment
- **We have two-stage development**: Safe development environment → Production-like testing

**What surprised me:**

- Uyuni has sophisticated build system that abstracts container deployment complexity
- **What is simpler than expected**: Once understood, the build/deploy process is straightforward
- **What is more complex than expected**: Multiple container concepts and permission models

---

## Next Steps

1. **Verify development environment**: Test full development cycle: edit → build → deploy → test
2. **Explore Salt event processor code related code:**  `SaltReactor.java`, `PGEventStream.java`, etc.
3. **Sort dependencies**: Identify what is the dependency of salt event processor, how to decouple the service

**Investigation needed:**

- How Salt events currently flow through Uyuni system
- Database for event processing
- Communication patterns between Salt master and minion
- Container networking requirements for extracted service
