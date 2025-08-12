# OpenSUSE Uyuni Salt Event Processor Extraction and Containerization (Pending)

This project focuses on extracting and containerizing the Salt event processor from Uyuni's Tomcat server into a standalone containerized service, improving performance and resource management.

## Background

Uyuni's Salt event processor runs within Tomcat alongside the web application, which can cause performance bottlenecks during high event processing loads. This project separates the Salt event processing logic into a dedicated containerized service that maintains the same functionality while improving system architecture.

## Project Status

This is a GSoC 2024 project implementing the extraction and containerization of Uyuni's Salt event processor. The project follows a phased approach from research and extraction to containerization and deployment integration.

### Current Phase: Implementation & Testing
- [x] **Phase 0-1**: Environment setup and analysis (Weeks 0-2)
- [x] **Phase 2**: Salt event processor extraction (Weeks 3-5)
- [x] **Phase 3**: Container implementation and Uyuni-tools integration (Weeks 6-8)
- [x] **Phase 4**: Testing and optimization (Weeks 9-11)

## Implementation
- [PR#10493](https://github.com/uyuni-project/uyuni/pull/10493)
- [PR#626](https://github.com/uyuni-project/uyuni-tools/pull/626)

## Documentation (Pending)

### Setup & Environment
- [Uyuni Development Environment Setup](doc/Uyuni-Development-Environment-Setup.md)
- [Uyuni Server Production Environment Setup](doc/Uyuni-server-production-environment-setup.md)

### Salt Event Processor Extraction Implementation and Testing Details
- [Salt Event Processor Extraction Implementation](doc/Salt-Event-Processor-Extraction-Implementation.md)
- [Salt Event Processor Extraction Initial Testing](doc/Salt-Event-Processor-Extraction-Initial-Testing.md)

### Containerization and deployment 
- [Salt Event Processor Container Implementation](doc/Salt-Event-Processor-Container-Implementation.md)
- [Salt Event Processor Image Build and Test](doc/Salt-Event-Processor-Image-Build-and-Test.md)

### Uyuni-tools Integration (Pending)


### Additional Resources
- Troubleshooting logs (pending upload)

## Key Features

- **Standalone Service**: Extracted Salt event processor maintains full functionality
- **Database Integration**: Connects to PostgreSQL for event processing
- **Container Support**: Podman-based containerization
- **Uyuni-tools Integration**: Deployment and configuration management
- **Health Monitoring**: Service health checks and monitoring endpoints

## Architecture (Pending)

The solution transforms the current integrated architecture into a microservices approach:
- **Before**: Salt events processed within Tomcat alongside web application
- **After**: Dedicated containerized service handling Salt event processing independently

## Contributing

This project is part of the openSUSE GSoC 2025 program. For questions or contributions, please refer to the Uyuni project community channels.

## Related Links

- [Original GSoC Issue](https://github.com/openSUSE/mentoring/issues/210)
- [Uyuni Project](https://github.com/uyuni-project/uyuni)
- [Uyuni-tools](https://github.com/uyuni-project/uyuni-tools)