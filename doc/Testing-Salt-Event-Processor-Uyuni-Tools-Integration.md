# Testing Salt Event Processor Integration with Uyuni-Tools

**Goal**: Verify the Event Processor container integration in Uyuni-tools works correctly across all deployment scenarios and lifecycle operations.

**Context**: After implementing the container integration into uyuni-tools, testing is required to ensure the Event Processor service deploys correctly.

**Approach**: Build and test uyuni-tools with the integrated changes, then verify functionality across all supported operations systematically using the built Event Processor image from OBS.

## Build and Setup Testing Environment

We need:

- Uyuni development environment with podman installed
- Access to the containerized Salt Event Processor image
- Built uyuni-tools with Event Processor integration
- Clean testing environment (no existing Uyuni installation)
1. **Container Image to be tested**

     `registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest`

2. **Prepare Build Environment**

    ```bash
    # Ensure Go modules are properly configured
    go mod download
    go mod verify
    go mod tidy  # Resolve any missing dependencies in container
    ```

3. **Build Uyuni-Tools with Integration**

    ```bash
    server:~/Documents/uyuni-tools> sudo ./build.sh
    ```

    We successfully get a clean build without errors.

4. **Verify Flag Integration**

    ```bash
    # Test that Event Processor flags appear in help output
    sudo ./bin/mgradm install podman --help
    ```

    **Expected Output**:

    ```
    ...
    Event Processor Flags:
          --eventprocessor-image string   Image for Event processor container (default "registry.opensuse.org/uyuni/server-salt-event-processor")
          --eventprocessor-tag string     Tag for Event processor container, overrides the global value if set
    ...
    ```

    Flags are properly registered and display correct default values.


## Core Functionality Testing

1. **Clean Environment: We need to clean the installed Uyuni server, before testing integration**

    ```bash
    # Clean any existing installation
    sudo mgradm uninstall --force --purge-volumes
    ```

2. **Install uyuni server with event processor image uisng the built code**

    ```bash
    # Install with Event Processor
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm -c mgradm.yaml install podman server.uyuni.lab \
      --eventprocessor-image registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest \
      --logLevel debug
    ```

    **Success indicator**:

    - Installation completes without errors
    - Event Processor container is deployed and running
    - Database connectivity established
3. **Check event process container status using integrated code**

    ```bash
    # Check overall system status
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm status
    ```

    **Expected Event Processor Entry**:

    ```
    ● uyuni-event-processor@0.service - Uyuni Event Processor Container
         Loaded: loaded (/etc/systemd/system/uyuni-event-processor@.service; enabled; preset: disabled)
        Drop-In: /etc/systemd/system/uyuni-event-processor@.service.d
                 └─generated.conf
         Active: active (running) since Wed 2025-07-30 11:51:01 EDT; 2min 50s ago
    ```

    **Detailed Service Check**:

    ```bash
    # Verify status of service
    systemctl status uyuni-event-processor@0.service

    # Verify container is running
    podman ps | grep event-processor

    # Check container logs
    journalctl -u uyuni-event-processor@0.service

    # Verify database connectivity
    systemctl | grep uyuni
    ```

    **Log shows:**

    - Container `uyuni-salt-event-processor-0` is `active (running)`
    - Network: Connected to `uyuni` network
    - Database: Environment variables properly configured
    - Logs: Java application startup successful

        ```bash
        Jul 30 11:51:05 server.uyuni.lab uyuni-salt-event-processor-0[23427]: 2025-07-30 15:51:05,765 [main] INFO  SaltService - Successfully connected to the Salt event bus
        ```


    We can conclude that the service is successfully running when using the integrated code to deploy the image.


## Lifecycle Operations Testing

1. **Stop/Start Operations**

    Stop uyuni server using integrated code

    ```bash
    # Stop Uyuni services
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm stop

    # Verify Event Processor stops
    systemctl status uyuni-event-processor@0.service
    ```

    Start uyuni server

    ```bash
    # Start Uyuni services
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm start

    # Verify Event Processor starts
    systemctl status uyuni-event-processor@0.service
    ```

    **Success indicators**:

    - Uyuni server stops/starts successfully with all services, including event processor.
    - Salt event bus reconnection successful
    - Database connections reestablished
    - No errors in service logs
2. **Restart Operation**

    ```bash
    # Restart all services
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm restart

    # Verify Event Processor restarts cleanly
    systemctl status uyuni-event-processor@0.service
    ```

    **Success indicators:**

    - Uyuni server restarts successfully with all services without error, including event processor.
    - Salt event bus reconnection successful
    - Database connections reestablished
    - No errors in service logs
3. **Upgrade Testing**

    ```bash
    # Install with released uyuni-tools first
    sudo mgradm -c mgradm.yaml install podman server.uyuni.lab

    # Upgrade using built uyuni-tools with Event Processor
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm upgrade podman \
      --eventprocessor-image registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest \
      --logLevel debug
    ```

    **Success indicators**:

    - Upgrade process completes successfully
    - Event Processor service deployed during upgrade
    - No existing services disrupted
    - Database connections reestablished
    - No errors in service logs
4. **Uninstall Testing**

    ```bash
    # Complete removal test
    server:~/Documents/uyuni-tools> sudo ./bin/mgradm uninstall --purge-volumes --force
    ```

    **Success indicators**:

    - Event Processor service stopped and removed
    - Container images cleaned up (if `-purge-images` used)
    - Systemd service files removed
    - No orphaned containers or volumes

## Performance and Monitoring

1. **Resource Usage Monitoring**

    ```bash
    # Monitor container resource usage
    podman stats uyuni-salt-event-processor-0

    # Check systemd resource limits
    systemctl status uyuni-event-processor@0.service
    ```

    **Monitor**: CPU usage, memory consumption, container restart frequency.


## Troubleshooting

## Test Results Summary

The Salt Event Processor integration with uyuni-tools has been successfully tested and demonstrates reliable operation across all major deployment scenarios. The implementation correctly follows Uyuni-tools patterns and supports production deployment.

### Integration Test Result

- Command and flags properly recognized
- Installation and uninstallation Integration works correctly, container deploys during installation
- Service Lifecycle management works correctly (Start/stop/restart/upgrade operations)
- System Integration: database connectivity established, Salt event bus connection successful
- Status and Monitoring works correctly, service appears in `mgradm status` output, systemd service shows correct states, and container logs is accessible

### Performance

- Startup Time: 1 minute for container initialization
- Memory Usage: Java application typical memory footprint
- Database Connections: Stable connection pool management
- Event Processing: Real-time event handling without significant latency