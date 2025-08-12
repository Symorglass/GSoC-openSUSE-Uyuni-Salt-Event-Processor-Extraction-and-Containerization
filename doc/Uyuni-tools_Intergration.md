# Uyuni-tools Integration 
 

## Service Installation Integration 

```go
mgradm install podman .... 
```

### Step 1: Create the Service Flags Structure

```go
// SaltEventProcessorFlags contains settings for Salt event processor container.
type SaltEventProcessorFlags struct {
	Replicas  int
	Image     types.ImageFlags `mapstructure:",squash"`
	IsChanged bool
}
```

### Step 2: Add Flags to ServerFlags

mgradm/shared/utils/flags.go

```go
// ServerFlags is a structure hosting the parameters for installation, migration and upgrade.
type ServerFlags struct {
	Image        types.ImageFlags `mapstructure:",squash"`
	Coco         CocoFlags
	Mirror       string
	HubXmlrpc    HubXmlrpcFlags
	Migration    MigrationFlags    `mapstructure:",squash"`
	Installation InstallationFlags `mapstructure:",squash"`
	// DBUpgradeImage is the image to use to perform the database upgrade.
	DBUpgradeImage     types.ImageFlags `mapstructure:"dbupgrade"`
	Saline             SalineFlags
	SaltEventProcessor SaltEventProcessorFlags `mapstructure:",squash"
	Pgsql              types.PgsqlFlags
}
```

### Step3: Create Flag Registration Functions

mgradm/shared/utils/cmd_utils.go

```go
// AddContainerImageFlags add container image flags to command.
func AddContainerImageFlags(
	cmd *cobra.Command,
	container string,
	displayName string,
	groupName string,
	imageName string,
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

// AddSaltEventProcessorFlag adds the Salt event processor related parameters to cmd.
func AddSaltEventProcessorFlag(cmd *cobra.Command) {
	_ = utils.AddFlagHelpGroup(cmd, &utils.Group{ID: "saltevent-container", Title: L("Salt Event Processor Flags")})
	AddContainerImageFlags(cmd, "saltevent", L("Salt event processor"), "saltevent-container", "server-salt-event-processor") ❓ 检查 image 是否此名字
	cmd.Flags().Int("saltevent-replicas", 0, L("How many replicas of the Salt event processor container should be started"))
	_ = utils.AddFlagToHelpGroupID(cmd, "saltevent-replicas", "saltevent-container")
}
```

### Step 4: Register Service in Service Registry

`shared/utils/uyuniservices.go`

```go
// UyuniServices is the list of services to expose.
var UyuniServices = []types.UyuniService{
  ... 
	{Name: "uyuni-server-salt-event-processor",
		Image: 		 SaltEventProcessorImage,
		Description: L("Salt event processor"),
		Replicas:    types.SingleOptionalReplica,
		Options: 	 []types.UyuniServiceOption{}},
	...
}

// SaltEventProcessorImage holds the flags to tune the Salt event processor container image.
var SaltEventProcessorImage = types.ImageFlags{
	Name:       "server-salt-event-processor",
	Tag:        DefaultTag,
	Registry:   DefaultRegistry,
	PullPolicy: DefaultPullPolicy,
}
```

### Step 6: Add Flags to Install Command & Step 7: Update Flag Change Detection

mgradm/cmd/install/podman/podman.go

```go
func newCmd(globalFlags *types.GlobalFlags, **run** utils.CommandFunc[podmanInstallFlags]) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "podman [fqdn]",
		Short: L("Install a new server on podman"),
		Long: L(`Install a new server on podman

The install podman command assumes podman is installed locally.

NOTE: installing on a remote podman is not supported yet!
`),
		Args: cobra.MaximumNArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			var flags podmanInstallFlags
			flagsUpdater := func(v *viper.Viper) {
				flags.ServerFlags.Coco.IsChanged = v.IsSet("coco.replicas")
				flags.ServerFlags.HubXmlrpc.IsChanged = v.IsSet("hubxmlrpc.replicas")
				flags.ServerFlags.Saline.IsChanged = v.IsSet("saline.replicas") || v.IsSet("saline.port")
				flags.ServerFlags.SaltEventProcessor.IsChanged = v.IsSet("saltevent.replicas")     // Step 7: Update Flag Change Detection
			}
			return utils.CommandHelper(globalFlags, cmd, args, &flags, flagsUpdater, **run**)
		},
	}

	adm_utils.AddMirrorFlag(cmd)
	shared.AddInstallFlags(cmd)    // 👇 Step 6: Add Flags to Install Command
	podman.AddPodmanArgFlag(cmd)

	return cmd
}

// NewCommand for podman installation.
func NewCommand(globalFlags *types.GlobalFlags) *cobra.Command {
	return newCmd(globalFlags, **installForPodman**)     //👇 this is calling newCmd and parse in installForPodman (which is then parsed in utils.CommandHelper for execution )
}

```

mgradm/cmd/install/shared/flags.go

```go
// AddInstallFlags add flags to install command.
func AddInstallFlags(cmd *cobra.Command) {

	...
	cmd_utils.AddCocoFlag(cmd)

	cmd_utils.AddHubXmlrpcFlags(cmd)

	cmd_utils.AddSalineFlag(cmd)
	
	cmd_utils.AddSaltEventProcessorFlag(cmd)
	...

}
```

mgradm/cmd/install/podman/utils.g     ❓需要检查

```go
func installForPodman(
	_ *types.GlobalFlags,
	flags *podmanInstallFlags,
	cmd *cobra.Command,
	args []string,
) error {

...

	if err := saltEventProcessor.SetupSaltEventProcessorContainer(
		systemd, authFile, flags.Image.Registry, flags.SaltEventProcessor, flags.Image, flags.Installation.DB,
	); err != nil { return err }

...
}
```

mgradm/shared/saltEventProcessor/saltEventProcessor.go

```go
func SetupSaltEventProcessorContainer(
	systemd podman.Systemd,
	authFile string,
	registry string,
	saltEventProcessorFlags adm_utils.SaltEventProcessorFlags,
	baseImage types.ImageFlags,
	db adm_utils.DBFlags,
) error {
	if err := writeSaltEventProcessorFiles(
		systemd, authFile, registry, saltEventProcessorFlags, baseImage, db,
	); err != nil {
		return err
	}
	return systemd.ScaleService(saltEventProcessorFlags.Replicas, podman.SaltEventProcessorService) // 👇
}
```

Add Service Constant

`shared/podman/systemd.go`:

```go
// SaltEventProcessorService is the name of the systemd service for the Salt Event Processor container.
const SaltEventProcessorService = "uyuni-salt-event-processor"
```

### Step 8: Create Service Template

Create `mgradm/shared/templates/saltEventProcessorServiceTemplate.go`:

```go
package templates

import (
	"github.com/uyuni-project/uyuni-tools/shared/types"
	"io"
	"text/template"
)

// SaltEventProcessorServiceTemplateData represents the data for salt event processor service.
const saltEventProcessorServiceTemplate = `
[Unit]
Description=Uyuni Salt Event Processor Container
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
    -e db_name={{ .DBName }} \
	{{- range .DBPort }}
	-e db_port={{ .Port }} \
	{{- end }}
    -e db_host={{ .DBHost }} \
    --secret={{ .DBUserSecret }},type=env,target=db_user \
    --secret={{ .DBPassSecret }},type=env,target=db_password \
    --replace \
    --name {{ .NamePrefix }}-salt-event-processor-%i \
    --hostname {{ .NamePrefix }}-server-event-processor-%i.mgr.internal \
    --network {{ .Network }} \
    ${UYUNI_SALT_EVENT_PROCESSOR_IMAGE}'
ExecStop=/usr/bin/podman stop --ignore -t 10 --cidfile=%t/%n-%i.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore -t 10 --cidfile=%t/%n-%i.ctr-id
PIDFile=%t/uyuni-server-event-processor-%i.pid
TimeoutStopSec=60
TimeoutStartSec=60
Type=forking

[Install]
WantedBy=multi-user.target default.target
`

type EventProcessorServiceTemplateData struct {
	NamePrefix   string          // "uyuni"
	Network      string          // "uyuni-server"
	DBUserSecret string          // "uyuni-db-user"
	DBPassSecret string          // "uyuni-db-pass"
	DBName       string          // "susemanager"
	DBPort       []types.PortMap //  5432
	//DBBackend    string // "postgresql"
	DBHost string // "db"
}

// Render will create the systemd configuration file.
func (data EventProcessorServiceTemplateData) Render(wr io.Writer) error {
	t := template.Must(template.New("service").Parse(saltEventProcessorServiceTemplate))
	return t.Execute(wr, data)
}

```


# Service Life Cycle Management Integration

## mgradm start podman

**mgradm/shared/podman/startstop.go**

- Add systemd.StartInstantiated(podman.EventProcessorService) to StartServices()

```go
func StartServices() error {
	return utils.JoinErrors(
		systemd.StartService(podman.DBService),
		systemd.StartInstantiated(podman.ServerAttestationService),
		systemd.StartInstantiated(podman.EventProcessorService),
		systemd.StartInstantiated(podman.HubXmlrpcService),
		systemd.StartInstantiated(podman.SalineService),
		systemd.StartService(podman.ServerService),
	)
}
```

## mgradm stop podman

**mgradm/shared/podman/startstop.go**

- Add systemd.StopInstantiated(podman.EventProcessorService) to StopServices()

```go
func StopServices() error {
	return utils.JoinErrors(
		systemd.StopInstantiated(podman.ServerAttestationService),
		systemd.StopInstantiated(podman.EventProcessorService),
		systemd.StopInstantiated(podman.HubXmlrpcService),
		systemd.StopInstantiated(podman.SalineService),
		systemd.StopService(podman.ServerService),
		systemd.StopService(podman.DBService),
	)
}
```

## mgradm restart podman

**mgradm/cmd/restart/podman.go**

- Add systemd.RestartInstantiated(podman.EventProcessorService) to restart logic

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
		systemd.RestartInstantiated(podman.EventProcessorService),
	)
}
```

## mgradm status podman

**mgradm/cmd/status/podman.go**

- Add event processor status check loop following the ServerAttestationService pattern

```go

	_ = utils.RunCmdStdMapping(
		zerolog.DebugLevel, "systemctl", "status", "--no-pager", fmt.Sprintf("%s@%d", podman.EventProcessorService, 0), // refer to ScaleService // TODO: check if follow the pattern, or enforce 1 here
	)
```

## mgradm uninstall podman

4. **mgradm/cmd/uninstall/podman.go**

- Add systemd.UninstallInstantiatedService(podman.EventProcessorService, !flags.Force)

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
		podman.GetServiceImage(podman.EventProcessorService + "@"),
		podman.GetServiceImage(podman.SalineService + "@"),
		podman.GetServiceImage(podman.DBService),
	}

	// Uninstall the service
	systemd.UninstallService("uyuni-server", !flags.Force)
	// Force stop the pod
	podman.DeleteContainer(podman.ServerContainerName, !flags.Force)
	systemd.UninstallInstantiatedService(podman.ServerAttestationService, !flags.Force)
	systemd.UninstallInstantiatedService(podman.HubXmlrpcService, !flags.Force)
	systemd.UninstallInstantiatedService(podman.EventProcessorService, !flags.Force)
	systemd.UninstallInstantiatedService(podman.SalineService, !flags.Force)
	systemd.UninstallService(podman.DBService, !flags.Force)
```

## mgradm upgrade podman

mgradm/shared/utils/cmd_utils.go

```go

// AddUpgradeEventProcessorFlag adds the event processor related parameters to cmd upgrade.
func AddUpgradeEventProcessorFlag(cmd *cobra.Command) {
	_ = utils.AddFlagHelpGroup(cmd, &utils.Group{ID: "event-processor", Title: L("Event Processor Flags")})
	AddContainerImageFlags(cmd, "eventprocessor", L("Event processor"), "event-processor", "server-salt-event-processor")
}
```

mgradm/cmd/upgrade/podman/podman.go

- Add event processor upgrade call in upgrade workflow

```go
type podmanUpgradeFlags struct {
	cmd_utils.ServerFlags `mapstructure:",squash"`
	Podman                podman.PodmanFlags
}

func newCmd(globalFlags *types.GlobalFlags, run utils.CommandFunc[podmanUpgradeFlags]) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "podman",
		Short: L("Upgrade a local server on podman"),
		Args:  cobra.ExactArgs(0),
		RunE: func(cmd *cobra.Command, args []string) error {
			var flags podmanUpgradeFlags
			flagsUpdater := func(v *viper.Viper) {
				flags.ServerFlags.Coco.IsChanged = v.IsSet("coco.replicas")
				flags.ServerFlags.HubXmlrpc.IsChanged = v.IsSet("hubxmlrpc.replicas")
				flags.ServerFlags.Saline.IsChanged = v.IsSet("saline.replicas") || v.IsSet("saline.port")
				flags.ServerFlags.EventProcessor.IsChanged = v.IsSet("eventprocessor.image") || v.IsSet("eventprocessor.tag") // TODO: check if needs to detect change of these two
			}
			return utils.CommandHelper(globalFlags, cmd, args, &flags, flagsUpdater, run)
		},
	}
	shared.AddUpgradeFlags(cmd)
	podman.AddPodmanArgFlag(cmd)
	return cmd
}
```

**mgradm/shared/eventProcessor/eventProcessor.go**

- Add Upgrade() function to handle event processor upgrades

```go
// Upgrade event processor
func Upgrade(
	systemd podman.Systemd,
	authFile string,
	registry string,
	eventProcessorFlags adm_utils.EventProcessorFlags,
	baseImage types.ImageFlags,
	db adm_utils.DBFlags,
) error {
	if eventProcessorFlags.Image.Name == "" {
		return nil
	}

	if err := writeEventProcessorFiles(
		systemd, authFile, registry, eventProcessorFlags, baseImage,
	); err != nil {
		return err
	}

	if !eventProcessorFlags.IsChanged {
		return systemd.RestartInstantiated(podman.EventProcessorService)
	}
	return systemd.ScaleService(1, podman.EventProcessorService) // TODO: we can't scale here with 1 replica, what to upgrade?
}
```