# Terraform-Internals
My official Learning Notes for Terraform Internals

## Table of Contents
1. [Terraform Architecture Overview](#terraform-architecture-overview)
2. [Core Components](#core-components)
3. [Execution Flow](#execution-flow)
4. [State Management](#state-management)
5. [Provider Plugins](#provider-plugins)
6. [Configuration Language](#configuration-language)
7. [Graph Creation and Traversal](#graph-creation-and-traversal)
8. [Module System](#module-system)
9. [Terraform Core Internals](#terraform-core-internals)
10. [Extending Terraform](#extending-terraform)
    
## Terraform Architecture Overview
Terraform follows a modular architecture organized around these main components:
- **Terraform CLI**: The command-line interface that users interact with
- **Terraform Core**: The engine that processes configurations and manages state
- **Provider Plugins**: External components that interact with specific platforms/services
- **State Backend**: Storage for infrastructure state information
- **Configuration Parser**: Processes HCL syntax into usable structures
```
┌───────────────┐     ┌───────────────────────┐     ┌───────────────┐
│               │     │                       │     │               │
│  Terraform    │────▶│      Terraform        │────▶│   Provider    │
│     CLI       │     │        Core           │     │   Plugins     │
│               │     │                       │     │               │
└───────────────┘     └───────────────────────┘     └───────────────┘
                               │     ▲                      │
                               │     │                      │
                               ▼     │                      ▼
                      ┌─────────────────────┐     ┌──────────────────┐
                      │                     │     │                  │
                      │  State Management   │     │  External APIs   │
                      │                     │     │                  │
                      └─────────────────────┘     └──────────────────┘
# Core Components

### 1. Terraform CLI

The CLI is the primary interface for users and supports commands like:

- `terraform init`: Initializes a working directory, downloads providers/modules
- `terraform plan`: Creates an execution plan
- `terraform apply`: Executes the plan to change infrastructure
- `terraform destroy`: Removes all resources managed by the configuration

**How It Works**: The CLI parses command-line arguments, validates inputs, and delegates functionality to Terraform Core.

### 2. Terraform Core

The heart of Terraform, responsible for:

- Configuration parsing and validation
- State management
- Dependency graph construction
- Plan creation and execution
- Provider plugin management

**Key Packages**:
- `terraform/`: Core package containing the main functionality
- `configs/`: Configuration loading and validation
- `states/`: State file management
- `plans/`: Plan creation and execution
- `providers/`: Provider plugin interface

### 3. Backend System

Manages where and how state is stored:

- **Local Backend**: Stores state in a local file (default)
- **Remote Backends**: AWS S3, Azure Blob Storage, GCS, Terraform Cloud, etc.

**Implementation**: Found in `backend/` package with specific backend implementations in subdirectories.

## Execution Flow

Terraform's execution follows these key phases:

### 1. Configuration Loading

```
Configuration Files (.tf) ──▶ HCL Parser ──▶ Configuration Object
```

**Implementation Details**:
- The `configs` package loads and parses HCL configurations
- Files are parsed into a configuration structure
- Variables are resolved and expressions are prepared for evaluation

### 2. State Loading

```
State Backend ──▶ State File Parser ──▶ State Object
```

**Implementation Details**:
- The current state is loaded from the configured backend
- State is parsed into memory as a structured object
- Lock mechanisms prevent concurrent modifications

### 3. Resource Graph Creation

```
Configuration ──▶ Graph Builder ──▶ Resource Graph
```

**Implementation Details**:
- Dependencies between resources are analyzed
- A directed graph is constructed with resources as nodes
- The graph represents the order of operations

### 4. Plan Generation

```
Resource Graph + Current State + Configuration ──▶ Plan
```

**Implementation Details**:
- Terraform compares desired configuration with current state
- It determines which resources need creation, updating, or deletion
- Creates a plan object containing all required changes

### 5. Plan Execution

```
Plan ──▶ Apply Engine ──▶ Provider Operations ──▶ New State
```

**Implementation Details**:
- The graph is traversed in the correct order
- Provider plugins are called to make actual infrastructure changes
- State is updated after each successful resource operation

## State Management

State is Terraform's source of truth about infrastructure:

### State Structure

```json
{
  "version": 4,
  "terraform_version": "1.0.0",
  "serial": 1,
  "lineage": "12345abc-def6-7890-1234-56789abcdef0",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "example",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-12345",
            "ami": "ami-12345",
            "instance_type": "t2.micro"
          },
          "private": "base64hash"
        }
      ]
    }
  ]
}
```

**Key Components**:
- **Version**: State file format version
- **Terraform Version**: Version that created the state
- **Serial**: Incremented on each state update
- **Lineage**: Unique ID for the state file
- **Resources**: List of managed resources and their attributes

### State Locking

To prevent concurrent modifications, Terraform implements state locking:

- **Lock Info**: Contains details about who holds the lock
- **Lock Implementation**: Backend-specific (e.g., DynamoDB for S3 backend)
- **Lock Timeout**: Configurable timeout for lock acquisition

## Provider Plugins

Providers are separate binaries that Terraform Core communicates with through gRPC:

### Provider Protocol

```
┌───────────────┐     ┌───────────────────────┐
│               │     │                       │
│  Terraform    │◀───▶│       Provider        │
│    Core       │     │       Plugin          │
│               │     │                       │
└───────────────┘     └───────────────────────┘
      gRPC                      │
                                ▼
                      ┌──────────────────────┐
                      │                      │
                      │    External APIs     │
                      │                      │
                      └──────────────────────┘
```

**Implementation Details**:
- Providers implement the Terraform Plugin Framework or SDK
- Communication occurs via the plugin protocol (gRPC)
- Each provider defines its own schema for resources and data sources

### Provider Discovery

Terraform finds providers through:
- Local plugin cache (`~/.terraform.d/plugins/` or equivalent)
- Plugin directory in the current working directory
- Terraform Registry (registry.terraform.io)

## Configuration Language

Terraform uses HashiCorp Configuration Language (HCL):

### Language Processing

```
HCL Files ──▶ Lexer ──▶ Parser ──▶ AST ──▶ Configuration Structure
```

**Key Components**:
- **Lexer**: Transforms text into tokens
- **Parser**: Converts tokens into an Abstract Syntax Tree (AST)
- **Evaluator**: Processes the AST into usable structures

### Expression Evaluation

Expressions are evaluated at different times:
- **During Configuration**: Basic syntax validation
- **During Planning**: When resolving references to existing resources
- **During Apply**: When computing values that depend on created resources

## Graph Creation and Traversal

The dependency graph is a core concept in Terraform:

### Graph Types

- **Resource Graph**: Shows dependencies between resources
- **Apply Graph**: Optimized graph for the apply phase
- **Destroy Graph**: Specialized graph for resource destruction (reverse dependencies)

### Graph Operations

- **Vertices**: Resources, data sources, providers, etc.
- **Edges**: Dependencies between vertices
- **Traversal**: Usually done in parallel where possible
- **Subgraphs**: Used for modules and other logical groupings

### Implementation

```go
// Simplified example of graph traversal
type Graph struct {
    Vertices []Vertex
    Edges    []Edge
}

func (g *Graph) Walk(callback func(Vertex) error) error {
    // Determine what vertices can be visited in parallel
    // Execute callback on each vertex in correct order
}
```

## Module System

Modules provide code reuse and encapsulation:

### Module Loading

```
┌───────────────┐     ┌───────────────────────┐     ┌───────────────┐
│               │     │                       │     │               │
│ Root Module   │────▶│   Module Installer    │────▶│ Child Modules │
│               │     │                       │     │               │
└───────────────┘     └───────────────────────┘     └───────────────┘
                                │
                                ▼
                      ┌──────────────────────┐
                      │                      │
                      │   Module Registry    │
                      │                      │
                      └──────────────────────┘
```

**Implementation Details**:
- Modules are stored in `.terraform/modules/`
- Each module has its own configuration structure
- Modules are loaded recursively with dependency resolution

### Module Communication

- **Input Variables**: Values passed from parent to child module
- **Output Values**: Results exported from child to parent module
- **Resource Instances**: Created within the module's scope

## Terraform Core Internals

Key internal packages and their functions:

### `terraform` Package

Central package containing core functionality:
- **Context**: Main execution context for operations
- **Evaluator**: Evaluates expressions in context
- **Graph**: Manages dependency graphs
- **NodeResourceInstance**: Represents a resource instance in the graph

### `configs` Package

Handles configuration loading and validation:
- **Parser**: Parses HCL into configuration structures
- **Module**: Represents a module's configuration
- **Variable**: Handles input variables
- **Provider**: Manages provider configurations

### `states` Package

Manages state representation:
- **State**: Main state structure
- **Module**: Module-specific state
- **ResourceInstance**: State for a specific resource instance
- **Backend**: Interface for state storage

## Extending Terraform

Ways to extend Terraform functionality:

### Custom Providers

Implement the Provider interface using the Terraform Plugin Framework:

```go
func Provider() provider.Provider {
    return &myProvider{
        // Implementation details
    }
}

type myProvider struct {
    // Provider configuration
}

func (p *myProvider) Schema() schema.Schema {
    return schema.Schema{
        // Define provider schema
    }
}

func (p *myProvider) Resources() []func() resource.Resource {
    return []func() resource.Resource{
        NewExampleResource,
    }
}

func (p *myProvider) DataSources() []func() datasource.DataSource {
    return []func() datasource.DataSource{
        NewExampleDataSource,
    }
}
```

### Remote State Backend

Implement the Backend interface to create a custom state backend:

```go
type Backend interface {
    ConfigSchema() *hcl.BodySchema
    Configure(config hcl.Body) error
    StateMgr(workspace string) (state.State, error)
}
```

### Function Extension

Custom functions can be added to Terraform through providers:

```go
func (p *Provider) Functions() []function.Function {
    return []function.Function{
        // Custom function implementations
    }
}
```

## Performance Considerations

Terraform's performance depends on several factors:

- **Graph Size**: More resources = larger graph = longer processing
- **Provider Operations**: API calls can be the bottleneck
- **State Size**: Large state files slow down operations
- **Parallelism**: Default is 10 concurrent operations (configurable)

## Debugging Terraform Internals

Useful techniques for debugging:

- **TF_LOG**: Set to TRACE, DEBUG, INFO, WARN, or ERROR
- **TF_LOG_PATH**: Output logs to a specific file
- **TF_LOG_CORE** and **TF_LOG_PROVIDER**: Specific component logging

Example:
```bash
export TF_LOG=TRACE
export TF_LOG_PATH=./terraform.log
terraform apply
```

## Best Practices for Terraform Development

When working with Terraform internals:

1. **Understand the Graph**: The dependency graph is fundamental
2. **State Management**: Always handle state with care
3. **Plugin Protocol**: Follow the protocol for provider development
4. **Expression Evaluation**: Consider when expressions are evaluated
5. **Testing**: Use the testing framework provided by Terraform

---
