# Salt Event Processor Uyuni-tools Integration Takeaways

**Context**: First experience with container deployment automation and Go-based CLI tools

**Accomplished**: Integrate a containerized Salt Event Processor service into uyuni-tools deployment pipeline following established patterns.

Integrated a new containerized service into uyuni-tools following established patterns.

## **Major Challenges Faced & My Navigation Strategy**

### **Challenge 1: Understanding the Architecture**

**Problem**: Initially overwhelmed by the complex layered architecture of uyuni-tools. Have zero experience implementing deployment tools in Golang.

**How I navigated**:

- Studied existing services (like, coco-attestation) to learn the implementation patterns
- Drew dependency diagrams to understand component relationships
- Traced data flow from CLI flags to running containers

**Key takeaway**: Complex systems become manageable when you narrow down the problem and find an element or thread to follow. By following how a particular event flows through the system, it will help you to understand the existing and save you from getting lost in complicated interdependencies.

### **Challenge 2: Go Language Patterns and CLI Framework Integration**

**Problem**: Unfamiliar with Go patterns used in enterprise level deployment workflow. Never worked with Cobra/Viper configuration frameworks.

**How I navigated**:

- Studied `Cobra` and `pflag` pattern in Go.
- Traced flag parsing flow from CLI to Go structs.
- Studied the template/text package for file generation

### **Challenge 3: Container Orchestration Concepts**

**Problem**: Don’t have in-depth understanding of systemd, container networking, and secret management

**Navigation Strategy**:

- Learn from solid Uyuni examples, learn by working
- Studied systemd instantiated services using the production example from uyuni-tools, learn how it is embedded in deployment management tools
- Learned about container networking and service discovery
- Understood the difference between templates and configuration

### **Challenge 4: Mentor Feedback Integration**

**Problem**: Managing project expectations while navigating implementation constraints and incorporating mentor feedback to adjust the technical approach.

**Navigation Strategy**:

- Gradually build up my understand of the event processing system, identify bottleneck and discuss the key implementation constraints with mentor to adjust the project architecture design. This helps effectively adjust the scope of deliverables to make the project more realistic given current system limitations.I
- Dive deep into domain-specific constraints by studying the codebase, make hypothesis then validate, explore codebase thinking about functional and non-function requirement of Uyuni to form a deeper understanding of the architectural decisions.
- Learned to question assumptions about scaling and design based on the actual user case, don’t over-complicate the scalability issue.

**Key Learning**: Not all services benefit from horizontal scaling.