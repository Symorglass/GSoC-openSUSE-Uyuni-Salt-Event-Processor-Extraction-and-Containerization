# Event Processor Uyuni-tools Integration - Complete Implementation

**Goal**: Integration of the extracted Salt Event Processor container into uyuni-tools deployment system, which enables service deployment alongside other Uyuni microservices.

**Context**: Salt event processing was extracted from Tomcat into a standalone containerized service. Now it needs full integration into uyuni-tools so users can deploy, manage, and scale it using standard uyuni-tools commands.

**Approach**: Follow the uyuni-tools microservice integration pattern of other services.

---

## What we built

**Event Processor Container Image:**

`registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest`

**Image was manually tested**:

```bash
# Run with production-like configuration
podman run -n salt-events-processor \
  -e db_name=susemanager \
  -e db_port=5432 \
  -e db_host=db \
  --secret uyuni-db-user,type=env,target=db_user \
  --secret uyuni-db-pass,type=env,target=db_password \
  --network=container:uyuni-server \
  registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest
```

## Integration Considerations

**Implementation Pattern**: Following established uyuni-tools microservice architecture like Coco-attestation (Confidential Computing), Saline, Hub XML-RPC

**Critical Decisions**:

- **Service Name**: Use `event-processor` instead of `salt-event-processor` , so service can be used by other event processors
- **Single replica enforcement:** We have 1 replica enforced (This is the initial version of event processing container, we can’t guarantee the event processing order with multiple replica, so it can't be distributed at this stage)
- **Database Backend**: Hardcoded to `postgresql`
- **Template Pattern**: Uses systemd instantiated service pattern (`service@0.service`)

## Step-by-Step Microservice Integration in Uyuni-tools

### Step 1: Create Service Flags Structure

**File**: `mgradm/shared/utils/types.go`

```go
// EventProcessorFlags contains settings for event processor container.
type EventProcessorFlags struct {
    Image     types.ImageFlags `mapstructure:",squash"`
    IsChanged bool
}
```

**Note:** Unlike other services (Coco, HubXmlrpc, Saline), EventProcessor does not have `Replicas int` field because it's enforced to exactly 1. Current Salt event processing was designed singleton - multiple instances processing the same event stream would cause race conditions. To implement replicas, we need to redesign the salt event processor logic.

### Step 2: Add Flags to ServerFlags

**File**: `mgradm/shared/utils/flags.go`

```go
// ServerFlags is a structure hosting the parameters for installation, migration and upgrade.
type ServerFlags struct {
    Image        types.ImageFlags `mapstructure:",squash"`
    Coco         CocoFlags
    Mirror       string
    HubXmlrpc    HubXmlrpcFlags
    Migration    MigrationFlags    `mapstructure:",squash"`
    Installation InstallationFlags `mapstructure:",squash"`
    DBUpgradeImage types.ImageFlags `mapstructure:"dbupgrade"`
    Saline         SalineFlags
    EventProcessor EventProcessorFlags `mapstructure:"eventprocessor"`  // Added this line
    Pgsql          types.PgsqlFlags
}
```

**Mapstructure Tag**: `eventprocessor` - this becomes the CLI flag prefix

### Step 3: Flag Registration Functions

**File**: `mgradm/shared/utils/cmd_utils.go`

**Container Image Flags Pattern**:

```go
func AddContainerImageFlags(
    cmd *cobra.Command,
    container string,        // "eventprocessor"
    displayName string,      // "Event processor"
    groupName string,        // "event-processor"
    imageName string,        // "server-salt-event-processor"
) {
    defaultImage := path.Join(utils.DefaultRegistry, imageName)
    cmd.Flags().String(container+"-image", defaultImage,
        fmt.Sprintf(L("Image for %s container"), displayName))
    cmd.Flags().String(container+"-tag", "",
        fmt.Sprintf(L("Tag for %s container, overrides the global value if set"), displayName))

    if groupName != "" {
        _ = utils.AddFlagToHelpGroupID(cmd, container+"-image", groupName)
        _ = utils.AddFlagToHelpGroupID(cmd, container+"-tag", groupName)
    }
}
```

**Event Processor Specific Functions**:

```go
// AddEventProcessorFlag adds the event processor related parameters to cmd.
func AddEventProcessorFlag(cmd *cobra.Command) {
    _ = utils.AddFlagHelpGroup(cmd, &utils.Group{ID: "event-processor", Title: L("Event Processor Flags")})
    AddContainerImageFlags(cmd, "eventprocessor", L("Event processor"), "event-processor", "server-salt-event-processor")
}

// AddUpgradeEventProcessorFlag adds the event processor related parameters to cmd upgrade.
func AddUpgradeEventProcessorFlag(cmd *cobra.Command) {
    _ = utils.AddFlagHelpGroup(cmd, &utils.Group{ID: "event-processor", Title: L("Event Processor Flags")})
    AddContainerImageFlags(cmd, "eventprocessor", L("Event processor"), "event-processor", "server-salt-event-processor")
}
```

We use `event-processor` instead of `salt-event-processor` as the service name, so this integration can be used by other event processors

### Step 4: Register Service in Service Registry

**File**: `shared/utils/uyuniservices.go`

```go
// UyuniServices is the list of services to expose.
var UyuniServices = []types.UyuniService{
    // ... 
    {Name: "uyuni-server-salt-event-processor",
     Image:       SaltEventProcessorImage,
     Description: L("Salt event processor"),
     Replicas:    types.SingleOptionalReplica,
     Options:     []types.UyuniServiceOption{}},
    // ... 
}

// SaltEventProcessorImage holds the flags to tune the Salt event processor container image.
var SaltEventProcessorImage = types.ImageFlags{
    Name:       "server-salt-event-processor",
    Tag:        DefaultTag,
    Registry:   DefaultRegistry,
    PullPolicy: DefaultPullPolicy,
}
```

### Step 5: Systemd Service Constants

**File**: `shared/podman/systemd.go`

```go
// EventProcessorService is the name of the systemd service for the Salt Event Processor container.
const EventProcessorService = "uyuni-event-processor"
```

### Step 6: Install Command Integration

**File**: `mgradm/cmd/install/shared/flags.go`

```go
// AddInstallFlags add flags to install command.
func AddInstallFlags(cmd *cobra.Command) {
    // ... other flags ...
    cmd_utils.AddCocoFlag(cmd)
    cmd_utils.AddHubXmlrpcFlags(cmd)
    cmd_utils.AddSalineFlag(cmd)
    cmd_utils.AddEventProcessorFlag(cmd)  // Added this line
    // ... more flags ...
}
```

**File**: `mgradm/cmd/install/podman/utils.go`

```go
func installForPodman(
    _ *types.GlobalFlags,
    flags *podmanInstallFlags,
    cmd *cobra.Command,
    args []string,
) error {
    // ... setup code ...

    if err := eventProcessor.SetupEventProcessorContainer(
        systemd, authFile, flags.Image.Registry, flags.EventProcessor, flags.Image, flags.Installation.DB,
    ); err != nil {
        return err
    }

    // ... more setup ...
}

```

### Step 7: Event Processor Service Implementation

**File**: `mgradm/shared/eventProcessor/eventProcessor.go`

**Setup Function**:

```go
func SetupEventProcessorContainer(
    systemd podman.Systemd,
    authFile string,
    registry string,
    eventProcessorFlags adm_utils.EventProcessorFlags,
    baseImage types.ImageFlags,
    db adm_utils.DBFlags,
) error {
    if err := writeEventProcessorFiles(
        systemd, authFile, registry, eventProcessorFlags, baseImage, db,
    ); err != nil {
        return err
    }
    // Enforce one replica
    return systemd.ScaleService(1, podman.EventProcessorService)
}
```

**Core Implementation Logic**:

```go
func writeEventProcessorFiles(
    systemd podman.Systemd,
    authFile string,
    registry string,
    eventProcessorFlags adm_utils.EventProcessorFlags,
    baseImage types.ImageFlags,
    db adm_utils.DBFlags,
) error {
    image := eventProcessorFlags.Image

    log.Debug().Msgf("Current running event processor replica is enforced to be 1")

    if !eventProcessorFlags.IsChanged {
        log.Debug().Msgf("Event processor settings are not changed.")
        return nil
    }

    // Always enable pulling if service is requested (since we enforced single replica)
    preparedImage, err := podman.PrepareImage(authFile, eventProcessorImage, baseImage.PullPolicy, true)
    if err != nil {
        return err
    }

    eventProcessorData := templates.EventProcessorServiceTemplateData{
        NamePrefix:   "uyuni",
        Network:      podman.UyuniNetwork,
        DBUserSecret: podman.DBUserSecret,
        DBPassSecret: podman.DBPassSecret,
        DBBackend:    "postgres",
    }

    log.Info().Msg(L("Setting up event processor service"))

    if err := utils.WriteTemplateToFile(
        eventProcessorData, podman.GetServicePath(podman.EventProcessorService+"@"), 0555, true,
    ); err != nil {
        return utils.Errorf(err, L("Failed to generate systemd service unit file"))
    }

    environment := fmt.Sprintf(`Environment=UYUNI_EVENT_PROCESSOR_IMAGE=%s
Environment=UYUNI_DB_NAME=%s
Environment=UYUNI_DB_PORT=%d
Environment=UYUNI_DB_HOST=%s`,
        preparedImage, db.Name, db.Port, db.Host)

    if err := podman.GenerateSystemdConfFile(
        podman.EventProcessorService+"@", "generated.conf", environment, true,
    ); err != nil {
        return utils.Errorf(err, L("cannot generate systemd user configuration file"))
    }

    if err := systemd.ReloadDaemon(false); err != nil {
        return err
    }

    return nil
}

```

**Upgrade Logic**:

```go
func Upgrade(
    systemd podman.Systemd,
    authFile string,
    registry string,
    eventProcessorFlags adm_utils.EventProcessorFlags,
    baseImage types.ImageFlags,
    db adm_utils.DBFlags,
) error {
    if eventProcessorFlags.Image.Name == "" {
        return fmt.Errorf(L("image is required"))
    }

    if err := writeEventProcessorFiles(
        systemd, authFile, registry, eventProcessorFlags, baseImage, db,
    ); err != nil {
        return err
    }

    return systemd.ScaleService(1, podman.EventProcessorService)
}

```

### Step 8: Systemd Service Template

**File**: `mgradm/shared/templates/saltEventProcessorServiceTemplate.go`

```go
const saltEventProcessorServiceTemplate = `
[Unit]
Description=Uyuni Event Processor Container
Wants=network.target
After=network-online.target

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
ExecStartPre=/bin/rm -f %t/uyuni-salt-event-processor-%i.pid %t/%n.ctr-id
ExecStartPre=/usr/bin/podman rm --ignore --force -t 10 {{ .NamePrefix }}-salt-event-processor-%i
ExecStart=/bin/sh -c '/usr/bin/podman run \
    --conmon-pidfile %t/uyuni-salt-event-processor-%i.pid \
    --cidfile=%t/%n-%i.ctr-id \
    --cgroups=no-conmon \
    --sdnotify=conmon \
    -d \
    -e db_name=${UYUNI_DB_NAME} \
    -e db_port=${UYUNI_DB_PORT} \
    -e db_host=${UYUNI_DB_HOST} \
    -e db_backend=postgresql \
    --secret={{ .DBUserSecret }},type=env,target=db_user \
    --secret={{ .DBPassSecret }},type=env,target=db_password \
    --replace \
    --name {{ .NamePrefix }}-salt-event-processor-%i \
    --hostname {{ .NamePrefix }}-server-event-processor-%i.mgr.internal \
    --network {{ .Network }} \
    ${UYUNI_EVENT_PROCESSOR_IMAGE}'
ExecStop=/usr/bin/podman stop --ignore -t 10 --cidfile=%t/%n-%i.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore -t 10 --cidfile=%t/%n-%i.ctr-id
PIDFile=%t/uyuni-salt-event-processor-%i.pid
TimeoutStopSec=60
TimeoutStartSec=60
Type=forking

[Install]
WantedBy=multi-user.target default.target
`

type EventProcessorServiceTemplateData struct {
    NamePrefix   string // "uyuni"
    Image        string
    Network      string // "uyuni-server"
    DBUserSecret string // "uyuni-db-user"
    DBPassSecret string // "uyuni-db-pass"
    DBBackend    string // "postgresql"
}

// Render will create the systemd configuration file.
func (data EventProcessorServiceTemplateData) Render(wr io.Writer) error {
    t := template.Must(template.New("service").Parse(saltEventProcessorServiceTemplate))
    return t.Execute(wr, data)
}

```

### Step 9: Lifecycle Operations Integration

**Start Services** (`mgradm/shared/podman/startstop.go`):

```go
func StartServices() error {
    return utils.JoinErrors(
        systemd.StartService(podman.DBService),
        systemd.StartInstantiated(podman.ServerAttestationService),
        systemd.StartInstantiated(podman.EventProcessorService),  // Added this line
        systemd.StartInstantiated(podman.HubXmlrpcService),
        systemd.StartInstantiated(podman.SalineService),
        systemd.StartService(podman.ServerService),
    )
}
```

**Stop Services**:

```go
func StopServices() error {
    return utils.JoinErrors(
        systemd.StopInstantiated(podman.ServerAttestationService),
        systemd.StopInstantiated(podman.EventProcessorService),  // Added this line
        systemd.StopInstantiated(podman.HubXmlrpcService),
        systemd.StopInstantiated(podman.SalineService),
        systemd.StopService(podman.ServerService),
        systemd.StopService(podman.DBService),
    )
}
```

**Restart Command** (`mgradm/cmd/restart/podman.go`):

```go
func podmanRestart(
    _ *types.GlobalFlags,
    _ *restartFlags,
    _ *cobra.Command,
    _ []string,
) error {
    return utils.JoinErrors(
        systemd.RestartService(podman.DBService),
        systemd.RestartService(podman.ServerService),
        systemd.RestartInstantiated(podman.ServerAttestationService),
        systemd.RestartInstantiated(podman.HubXmlrpcService),
        systemd.RestartInstantiated(podman.EventProcessorService),  // Added this line
    )
}
```

**Status Command** (`mgradm/cmd/status/podman.go`):

```go
func podmanStatus(
    _ *types.GlobalFlags,
    _ *statusFlags,
    _ *cobra.Command,
    _ []string,
) error {
    // ... other status checks ...

    _ = utils.RunCmdStdMapping(
        zerolog.DebugLevel, "systemctl", "status", "--no-pager",
        podman.EventProcessorService+"@0",  // Added this line - hardcoded to @0 since single replica
    )

    return nil
}
```

**Uninstall Command** (`mgradm/cmd/uninstall/podman.go`):

```go
func uninstallForPodman(
    _ *types.GlobalFlags,
    flags *utils.UninstallFlags,
    _ *cobra.Command,
    _ []string,
) error {
    // Get the images from the service configs before they are removed
    images := []string{
        podman.GetServiceImage(podman.ServerService),
        podman.GetServiceImage(podman.ServerAttestationService + "@"),
        podman.GetServiceImage(podman.HubXmlrpcService + "@"),
        podman.GetServiceImage(podman.EventProcessorService + "@"),  // Added this line
        podman.GetServiceImage(podman.SalineService + "@"),
        podman.GetServiceImage(podman.DBService),
    }

    // Uninstall the service
    systemd.UninstallService("uyuni-server", !flags.Force)
    // Force stop the pod
    podman.DeleteContainer(podman.ServerContainerName, !flags.Force)
    systemd.UninstallInstantiatedService(podman.ServerAttestationService, !flags.Force)
    systemd.UninstallInstantiatedService(podman.HubXmlrpcService, !flags.Force)
    systemd.UninstallInstantiatedService(podman.EventProcessorService, !flags.Force)  // Added this line
    systemd.UninstallInstantiatedService(podman.SalineService, !flags.Force)
    systemd.UninstallService(podman.DBService, !flags.Force)

    // ... volume and image cleanup ...
}
```

### Step 10: Upgrade Command Integration

**File**: `mgradm/cmd/upgrade/podman/podman.go`

```go
type podmanUpgradeFlags struct {
    cmd_utils.ServerFlags `mapstructure:",squash"`
    Podman                podman.PodmanFlags
}

func newCmd(globalFlags *types.GlobalFlags, run utils.CommandFunc[podmanUpgradeFlags]) *cobra.Command {
    cmd := &cobra.Command{
        // ... command setup ...
        RunE: func(cmd *cobra.Command, args []string) error {
            var flags podmanUpgradeFlags
            flagsUpdater := func(v *viper.Viper) {
                flags.ServerFlags.Coco.IsChanged = v.IsSet("coco.replicas")
                flags.ServerFlags.HubXmlrpc.IsChanged = v.IsSet("hubxmlrpc.replicas")
                flags.ServerFlags.Saline.IsChanged = v.IsSet("saline.replicas") || v.IsSet("saline.port")
                flags.ServerFlags.EventProcessor.IsChanged = v.IsSet("eventprocessor.image") || v.IsSet("eventprocessor.tag")  // Added this line
            }
            return utils.CommandHelper(globalFlags, cmd, args, &flags, flagsUpdater, run)
        },
    }
    shared.AddUpgradeFlags(cmd)
    podman.AddPodmanArgFlag(cmd)
    return cmd
}
```

**File**: `mgradm/cmd/upgrade/shared/flags.go`

```go
// AddUpgradeFlags add flags to upgrade command.
func AddUpgradeFlags(cmd *cobra.Command) {
    // ... other flags ...
    cmd_utils.AddUpgradeCocoFlag(cmd)
    cmd_utils.AddUpgradeHubXmlrpcFlags(cmd)
    cmd_utils.AddUpgradeSalineFlag(cmd)
    cmd_utils.AddUpgradeEventProcessorFlag(cmd)  // Added this line
    // ... more flags ...
}
```

**File**: `mgradm/cmd/upgrade/podman/utils.go`

```go
func upgradePodman(_ *types.GlobalFlags, flags *podmanUpgradeFlags, cmd *cobra.Command, _ []string) error {
    // ... setup code ...

    return podman.Upgrade(
        systemd, authFile,
        flags.Image.Registry,
        flags.Installation.DB,
        flags.Installation.ReportDB,
        flags.Installation.SSL,
        flags.Image,
        flags.DBUpgradeImage,
        flags.Coco,
        flags.HubXmlrpc,
        flags.Saline,
        flags.EventProcessor,  // Added this line
        flags.Pgsql,
        flags.Installation.SCC,
        flags.Installation.TZ,
    )
}
```

**File**: `mgradm/shared/podman/podman.go`

```go
func Upgrade(
    systemd podman.Systemd,
    authFile string,
    registry string,
    // ... other parameters ...
    eventProcessorFlags adm_utils.EventProcessorFlags,  // Added this parameter
    // ... more parameters ...
) error {
    // ... upgrade logic ...

    if err := eventProcessor.Upgrade(systemd, authFile, registry, eventProcessorFlags, image, inspectedDB); err != nil {
        return utils.Errorf(err, L("error upgrading event processor service."))
    }

    return systemd.ReloadDaemon(false)
}
```

---

## Current Implementation

**Integration Check:**

- [x]  Service flags structure
- [x]  ServerFlags integration with mapstructure
- [x]  Flag registration functions
- [x]  Service registry in UyuniServices
- [x]  Systemd service template
- [x]  Event processor service implementation
- [x]  Install command integration
- [x]  Upgrade command integration
- [x]  Start/stop/restart service integration
- [x]  Status command integration
- [x]  Uninstall command integration

## Design consideration

The scalability of event processor is limited by its architecture design.

- **Replica Enforcement Strategy**: EventProcessor does not have `Replicas int` field because it's enforced to exactly 1. Current Salt event processing was designed singleton, Current event processing order is guaranteed by single instance of processor.  Multiple instances processing the same event stream would cause race conditions. To implement replicas, we need to redesign the salt event processor logic
- **Single Point bottleneck:** This processor is now the single point for all Salt event processing, if bottleneck emerges, need event partitioning strategy

For more analysis, check: [Uyuni Salt Event Processing System and Uyuni-tools Integration: Design, and Scaling Considerations ](doc/Current-Design-and-Scaling-Considerations.md)

## **Future Consideration**

The event processing system can be horizontally scaled with proper event partitioning, but that's a major architectural change which requires designing the whole event processing logic using proper partitioning strategy. Vertical scaling is well-suited for Uyuni’s current use.

## Conclusion

This integration of event processing into Uyuni-tools enables the deployment of a standalone containerized event processing service. It follows the uyuni-tools microservice patterns. The architecture properly handles the singleton constraint of event processing while maintaining flexibility for future scaling scenarios. The use of systemd templates, podman secrets, and established uyuni-tools patterns provides a solid foundation for production deployment.