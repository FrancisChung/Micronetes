# Micronetes

Micronetes is a local orchestrator inspired by kubernetes that makes developing and testing microservices and distributed systems easier.

This project is broken into 2 loosely coupled components:
- **The Micronetes CLI** - This is the orchestrator used for development and testing.
- **The Micronetes SDK** - This adds a layer on top of the naming conventions introduced by the orchestor.

## Micronetes CLI

The Micronetes CLI is an orchestrator that coordinates multiple applications running both locally and remotely to make developing easier. The core model is an application which is made up of several services. 

```yaml
- name: web
  project: Web\Web.csproj
  replicas: 2
  bindings:
    - port: 5005
- name: worker
  project: Worker\Worker.csproj
  replicas: 3
- name: rabbitmq
  dockerImage: rabbitmq
  bindings:
    - port: 5672
```

**Reference**

```yaml
- name: string  # name of the service
  project: string  # msbuild project path (relative to this file)
  executable: string # path to an executable (relative to this file)
  workingDirectory: string # working directory of the process (relative to this file)
  args: string # arguments to pass to the process
  replicas: number # number of times to launch the application
  env: # environment variables
  bindings: # array of bindings (ports, connection strings etc)
    name: string # name of the binding
    port: number # port of the binding
    host: string # host of the binding
    connectionString: # connection string of the binding
```

### Service Descriptions

A service description is yaml file with list of services. Services can have multiple bindings that describe how the application can connect to it.

```yaml
- name: redis
  dockerImage: redis:5
  bindings:
    - port: 6379
      protocol: redis
```

Examples:

**Executable listening on port 80**

```yaml
- name: dapr
  executable: daprd
  bindings:
    - port: 80
      protocol: dapr
```

**HTTP(s)**

```yaml
- name: myweb
  projectFile: MyWeb/Web.csproj
  bindings:
    - name: default
      port: 80
      protocol: http
    - name: management
      port: 3000
      protocol: http
```

These service names are injected into the application as environment variables by the orchestrator. This allows the client code to access the address information at runtime.

The following binding:

```yaml
- name: myweb
  projectFile: MyWeb/Web.csproj
  bindings:
    - name: default
      port: 80
      protocol: http
    - name: management
      port: 3000
      protocol: http
```

Will be translated into the following `IConfiguration` keys:

```
myweb:service:port=80
myweb:service:protocol=http
myweb:service:management:port=3000
myweb:service:management:protocol=http
```

The "default" binding can be accesed by the name of the service. To access other addresses, the full name must be accessed:

e.g.

```C#
// Access the default port of the MyWeb service
IClientFactory<HttpClient> clientFactory = ...
var defaultClient = clientFactory.CreateClient("myweb");

// Access the management port of the MyWeb service
var managementClient = clientFactory.CreateClient("myweb/management");
```

Most services will have a single binding so accessing them by service name directly should work.

##  The Micronetes SDK

The Micronetes SDK uses the conventions introduced by the orchestrator and introduces primitives that simplify microservice communication. The core abstraction for communicating with another microservice is an `IClientFactory<TClient>`.

```C#
public interface IClientFactory<TClient>
{
    TClient CreateClient(string name);
}
```

### Client abstractions

- HTTP - `HttpClient`
- PubSub - `PubSubClient`
- Queue - TBD
- RPC - TBD

The intent is to make decouple service addresses from the implementation. There are 2 flavours of TClient:

1. TClient can be an abstraction. 
2. TClient can be a concrete client implementation (like ConnectionMultipler).
