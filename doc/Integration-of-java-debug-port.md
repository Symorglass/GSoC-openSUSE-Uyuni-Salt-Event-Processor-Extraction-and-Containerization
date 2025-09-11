# Integration of Port `--debug-java` Support for Salt Event Processor in Uyuni-Tools
**Goal**: Integrate Java debug server for the Salt Event Processor service, by adding `--debug-java` flag.

**Context**: The Salt Event Processor runs as a separate containerized service in Uyuni's architecture. To provide java debug port option for Salt Event Processor service, `--debug-java` flag should be integrated following the existing patterns.


## Overview: Two-Layer Debug Architecture

The debug integration requires both network-level and process-level configuration:

**Production Mode (debugJava = false)**

```bash
# Port loop generates: (nothing)
# JAVA_OPTS: ""
# Result: No external access, no internal debug server*
```

**Debug Mode (debugJava = true)**

```bash
# Port loop generates: -p 8004:8004
# JAVA_OPTS: "-Xdebug -Xrunjdwp:..."
# Result: External access enabled, internal debug server running*
```

# Locate debug port patterns

1. search jdwp
2. search port like 8003, 8002, etc.
3. search `flags.Installation.Debug.Java`

# Integrate jave debug port

## 1. create EventProcessorPorts

In `shared/utils/ports.go`

```go
	// EventProcessorServiceName is the name of the server event processing service
	EventProcessorServiceName = "eventprocessor"
```

create new portmap for event processor service

```go
var EventProcessorPorts = []types.PortMap{
	NewPortMap(EventProcessorServiceName, "debug", 8004, 8004),
}
```

## 2. Add Debug Port Support to Service Template

Update the systemd service template to include the port loop and `JAVA_OPTS` environment variable:

```go
package templates

import (
	"io"
	"text/template"

	"github.com/uyuni-project/uyuni-tools/shared/types"
)

// SaltEventProcessorServiceTemplateData represents the data for salt event processor service.
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
	  {{- range .Ports }}
        -p {{ .Exposed }}:{{ .Port }}{{if .Protocol}}/{{ .Protocol }}{{end}} \
    {{- end }}
    -e db_name=${UYUNI_DB_NAME} \
	  -e db_port=${UYUNI_DB_PORT} \
    -e db_host=${UYUNI_DB_HOST} \
		-e db_backend=postgresql \
    -e JAVA_OPTS="${JAVA_OPTS}" \
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
	NamePrefix   string
	Image        string
	Network      string
	DBUserSecret string
	DBPassSecret string
	DBBackend    string
	Ports        []types.PortMap
}

// Render will create the systemd configuration file.
func (data EventProcessorServiceTemplateData) Render(wr io.Writer) error {
	t := template.Must(template.New("service").Parse(saltEventProcessorServiceTemplate))
	return t.Execute(wr, data)
}

```

## 3. Implement systemd service file rendering

template will be rendered into systemd service file through: 

```go
// Control debug port expose from container to client
var ports []types.PortMap
if debugJava {
	ports = utils.EventProcessorPorts
}

eventProcessorData := templates.EventProcessorServiceTemplateData{
	NamePrefix:   "uyuni",
	Network:      podman.UyuniNetwork,
	DBUserSecret: podman.DBUserSecret,
	DBPassSecret: podman.DBPassSecret,
	DBBackend:    "postgres",
	Ports:        ports,
}

if err := utils.WriteTemplateToFile(
	eventProcessorData, podman.GetServicePath(podman.EventProcessorService+"@"), 0555, true,
); err != nil {
	return utils.Errorf(err, L("Failed to generate systemd service unit file"))
}
```

## 3. Add systemd configuration file

generate a systemd configuration file with conditional port

```go
	// Add conditional debug server inside container
	var javaOpts string
	if debugJava {
		javaOpts = "-Xdebug -Xrunjdwp:transport=dt_socket,address=*:8004,server=y,suspend=n"
	} else {
		javaOpts = ""
	}

	environment := fmt.Sprintf(`Environment=UYUNI_EVENT_PROCESSOR_IMAGE=%s
Environment=UYUNI_DB_NAME=%s
Environment=UYUNI_DB_PORT=%d
Environment=UYUNI_DB_HOST=%s
Environment=JAVA_OPTS="%s"`,
		preparedImage, db.Name, db.Port, db.Host, javaOpts)

	if err := podman.GenerateSystemdConfFile(
		podman.EventProcessorService+"@", "generated.conf", environment, true,
	); err != nil {
		return utils.Errorf(err, L("cannot generate systemd user configuration file"))
	}
```

# How template variables work?

## **Port Loop: Template-Time Data-Driven**

```
{{- range .Ports }}
-p {{ .Exposed }}:{{ .Port }}{{if .Protocol}}/{{ .Protocol }}{{end}} \
{{- end }}
```

**Purpose**: Controls whether external clients can **reach** the container from outside

- **With port loop**: Creates network path `host:8004 → container:8004`

**How it works:**

- Uses data from `EventProcessorServiceTemplateData.Ports` field
- Processed when **template is rendered** during uyuni-tools writes the systemd service file (during install/upgrade)
- Generates static port `-p` flags in the systemd service file
    
    my systemd template 
    
    ```bash
    ExecStart=/bin/sh -c '/usr/bin/podman run \
        ...
    	  {{- range .Ports }}
            -p {{ .Exposed }}:{{ .Port }}{{if .Protocol}}/{{ .Protocol }}{{end}} \
        {{- end }}
        ...
    ```
    
    after port loop, systemd template will be rendered into a systemd file with port mapped into the 8004 port
    
    ```bash
    ExecStart=/bin/sh -c '/usr/bin/podman run \
        ...
            -p 8004:8004 \
        ...
    ```
    

**Data flow**:

1. Go code sets `ports := utils.EventProcessorPorts`
2. Go code passes `Ports: ports` in template data
3. Template loops through `Ports` slice
4. Result: Static `p 8004:8004` in service file

## **Environment Variable: Runtime Systemd Substitution**

**How it works**:

- `podman.GenerateSystemdConfFile` generates a systemd configuration file `generated.conf` with the `JAVA_OPTS`environment variable
- At container runtime, systemd substitutes `${JAVA_OPTS}` with the actual environment value

**Data flow**:

1. Go code sets `Environment=JAVA_OPTS="..."` in `generated.conf`
2. Systemd substitutes `${JAVA_OPTS}` in template with environment value at runtime
3. Result: Dynamic `e JAVA_OPTS="actual_value"` when container starts

```
-e JAVA_OPTS="${JAVA_OPTS}"
```

**Purpose**: Controls whether Java process listens for debug connections inside the container

- **With `JAVA_OPTS`**: Java starts debug server on port 8004 inside container
- **Without `JAVA_OPTS`**: Java runs normally, no debug server

**How it works:**

- `podman.GenerateSystemdConfFile` generate a systemd configuration file `generated.conf` with the `JAVA_OPTS` environment variable :
    
    ```go
    	// Add conditional debug server inside container
    	var javaOpts string
    	if debugJava {
    		javaOpts = "-Xdebug -Xrunjdwp:transport=dt_socket,address=*:8004,server=y,suspend=n"
    	} else {
    		javaOpts = ""
    	}
    
    	environment := fmt.Sprintf(`Environment=UYUNI_EVENT_PROCESSOR_IMAGE=%s
    Environment=UYUNI_DB_NAME=%s
    Environment=UYUNI_DB_PORT=%d
    Environment=UYUNI_DB_HOST=%s
    Environment=JAVA_OPTS="%s"`,
    		preparedImage, db.Name, db.Port, db.Host, javaOpts)
    
    	if err := podman.GenerateSystemdConfFile(
    		podman.EventProcessorService+"@", "generated.conf", environment, true,
    	); err != nil {
    		return utils.Errorf(err, L("cannot generate systemd user configuration file"))
    	}
    ```
    
- At container runtime, systemd substitutes `${JAVA_OPTS}` with the actual environment value from systemd configuration file `generated.conf`
    
    ```bash
    server:/ # cat etc/systemd/system/uyuni-event-processor@.service.d/generated.conf 
    # This file is generated by mgradm and will be overwritten during upgrades.
    # Custom configuration should go in another .conf file in the same folder.
    
    [Service]
    Environment=UYUNI_EVENT_PROCESSOR_IMAGE=registry.opensuse.org/home/cbosdonnat/branches/systemsmanagement/uyuni/master-sywen/containerfile/uyuni/server-salt-event-processor:latest
    Environment=UYUNI_DB_NAME=susemanager
    Environment=UYUNI_DB_PORT=5432
    Environment=UYUNI_DB_HOST=db
    Environment=JAVA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=*:8004,server=y,suspend=n"
    ```
    

**Data flow:**

1. Go code: Sets `Environment=JAVA_OPTS="..."` in `generated.conf`
2. Systemd: Substitutes `${JAVA_OPTS}` in template with environment value at runtime
3. Result: Dynamic `e JAVA_OPTS="actual_value"` when container starts
    
    ```bash
    # generated.conf
    Environment=JAVA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=*:8004,server=y,suspend=n"
    
    # Runtime substitution in systemd service
    -e JAVA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=*:8004,server=y,suspend=n"
    
    ```
    

# Key thoughts

The debug functionality requires both port mapping and environment variable configuration:

If only has port mapping, no `JAVA_OPTS`:

- External debugger can connect to host:8004
- Connection reaches container port 8004
- Nothing listening inside container → connection refused

Only `JAVA_OPTS`, no port mapping:

- Java debug server running inside container on port 8004
- No external access to container → cannot connect from outside
- Only internal container connections work

Both layers must be configured for external debugging tools to successfully attach to the Java process running inside the event processor container.