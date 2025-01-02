---
KEP: 2024-0001
Title: Risk Engine externalisation
Authors: 
  - Thibault Normand <Zenithar>
Status: Draft
Created: 2024-12-23
Last Updated: 2025-01-02
Version: draft-2
---

# Abstract

KubeHound is a tool that helps to identify security risks in Kubernetes clusters. 
It is a static analysis tool that scans Kubernetes resources and manifests to 
identify potential security risks. KubeHound uses a risk engine to analyse the
Kubernetes resources and to determine the risk level of the resources. The current
implementation is focused on PermissionSet only. We want to extend this to other
resources.

This KEP proposes to externalise the risk engine in KubeHound to make it easier 
to add new resource-based checks and maintain the risk engine.

# Motivation

The current risk engine in KubeHound is tightly coupled with the core codebase, 
making it challenging to add dynamic decisions and maintain the risk engine. 

By externalising the risk engine, we can make it easier to add new checks and
maintain the risk engine, as well as to make it easier to extend the risk
engine to add external support data for decision-making.

# Proposal

The risk engine in KubeHound will be externalised into a separate module. The
current risk engine will be refactored to be used as the default risk engine to 
ensure backward compatibility.

# Design

Externalisation strategies will allow the risk engine to be easily extended to 
add new resource checks and to be easily maintained. Customers will be able to 
add their own custom checks to the risk engine by implementing a simple service 
or by adding new checks to the risk engine.

The risk engine will load checks from support material such as files and databases
and then run the checks against the Kubehound resources during the resource 
ingestion. The risk engine will return the risk analysis results to the core 
codebase for graph embedding.

## Risk Engine Decisions

The risk engine will be used to define:

- The resource's criticality (High, Medium, Low).
- The resource's compromise status (Compromised, Not Compromised).

These decisions will be embedded in the vertices and edges of the KubeHound 
graph to allow the core codebase to query the graph and to get the results based
on the risk analysis (e.g. critical paths from a compromised resource).

## Example

This section provides examples of how the risk engine will be used to determine
the risk of a Kubehound resource.

> These examples are illustrative and do not represent the final implementation.

### Pod Security Context

The risk engine will be able to check the pods' security context and determine 
if the security context is set correctly. The risk engine will be able to check 
the following:

- The pod is running as a privileged pod.
- The pod is running as a non-root pod.
- The pod has mounted sensitive volumes.

### Container Image Security

The risk engine will be able to check the security of the container images and
determine if the container images are secure. The risk engine will be able
to check the following:

- The container image has identified vulnerabilities.
- The container image is running with the latest version.

For performance reasons, the risk engine should consume the results of the image
and vulnerability scanning tools. Querying these tools inline
would be too slow. The scanning reports should be used as support material for
the risk engine.

### RBAC Permissions

The risk engine will be able to check the RBAC permissions and determine if the
permissions are set correctly. The risk engine will be able to check the following:

- The user is a privileged user
- The service account has excessive permissions

# Implementation(s)

The risk engine will be externalised in two ways:

- Embedded risk engine with external checks where the risk engine is embedded in
  the core codebase and the checks are externalised as separate configuration 
  items.
- Remote risk engine where the risk engine is externalised as a separate module
  communicating with the core codebase using gRPC streams.

These integrations will allow the risk engine to be adapted to the customer's
needs.

## Embedded risk engine with external checks

The risk engine will be embedded in the core codebase, and the checks will be
externalised as separate configuration items. 

The risk engine will be responsible for loading the checks from the configuration
file and running the checks against the Kubehound resources.

This approach will allow to keep the risk engine in the core codebase and to 
externalise the checks without the need to maintain a separate service.

The risk evaluation will be done in the core codebase.

### Sample checks configuration

> This is an illustrative example and does not represent the final implementation.

<details><summary>Critical Service Account</summary>

```yaml
apiVersion: kubehound.io/v1
kind: RiskEngineCheck
metadata:
  name: critical-service-account
spec:
  # The resource kind to check.
  kind: Identity
  # The conditions to filter the resources.
  filter:
    # Evict the resource if the type is not "serviceaccount".
    - key: "type"
      operator: eq
      value: "serviceaccount"
  # The checks to run against the resource.
  checks:
    # Name matches ".*-admin"
    - properties:
        - key: "name"
          operator: match
          value: ".*-admin"
    # Or name is "kube-system" or "default"
    - properties:
        - key: "name"
          operator: in
          value: ["kube-system", "default"]
  decision:
    # The risk level of the resource.
    riskLevel: high
```
</details>

<details><summary>Vulnerable Container</summary>

```yaml
apiVersion: kubehound.io/v1
kind: RiskEngineCheck
metadata:
  name: dvwa-vulnerable-container
spec:
  # The resource kind to check.
  kind: Container
  # The checks to run against the resource.
  checks:
    # The container image is DVWA.
    - properties:
      - key: "image"
        operator: match
        value: "citizenstig/dvwa.*"
  decision:
    # The risk level of the resource.
    riskLevel: critical
    # Flag the resource as compromised.
    compromised: true
```

</details>

The risk engine configuration will be stored in a configuration file handled by
the core codebase.

```yaml
# The configuration for the risk engine.
risk-engine:
  # The risk engine is embedded in the core codebase and the checks are externalised.
  checks:
    # Where the checks are stored.
    directory: /etc/kubehound/checks
```

### Evaluation

The proposed risk engine externalisation strategy has the following pros and cons:

| Evaluation | Comment |
|------------|---------|
| Pros       | - The risk engine is embedded in the core codebase. <br> - The checks are externalised as separate configuration items. <br> - The risk engine can be easily extended by adding new checks. |
| Cons       | - The risk engine evaluation are limited by the check expression language. <br> - Limited to static checks based on a predefined set of conditions. |

## Remote risk engine

The risk engine will be externalised as a separate module communicating with the
core codebase using gRPC streams. The risk engine will be responsible for 
responding to requests from the core codebase and returning the risk analysis 
results.

```
+-------------------+                     +-----------------+
|                   |                     |                 |
|   Core Codebase   | <-[ gRPC stream ]-> |   Risk Engine   |
|  << kubehound >>  |                     |  << process >>  |
|                   |                     |                 |
+-------------------+                     +-----------------+
```

The risk engine will be implemented in Go and packaged as a standalone
executable. The risk engine will be responsible for loading the checks from a
configuration file and running the checks against the Kubehound resources.

Using gRPC streams will allow the risk engine to handle many requests required 
for large clusters and define a contract between the core codebase and the risk 
engine. The implementer can build a risk engine using any language supporting gRPC.

### Protocol

The risk engine will communicate with the core codebase using gRPC bidirectional 
streams. The bidirectional streams will allow the core codebase to send requests
to the risk engine and to receive responses from the risk engine synchronously 
in near real-time.

The risk engine will expose the following gRPC service:

<details><summary>Risk Engine Protocol</summary>

```protobuf
// The RiskEngine service provides an interface for analysing the risk of
// Kubehound resources.
service RiskEngine {
  //Analyse the given Kubehound resource and return the risk analysis.
  rpc Analyse (stream AnalyzeRequest) returns (stream AnalyzeResponse) {}
}

// ResourceKind represents the Kubehound resource.
enum ResourceKind {
 RESOURCE_KIND_UNSPECIFIED = 0;
 RESOURCE_KIND_CONTAINER = 1;
 RESOURCE_KIND_GROUP = 2;
 RESOURCE_KIND_IDENTITY = 3;
 RESOURCE_KIND_NODE = 4;
 RESOURCE_KIND_PERMISSION_SET = 5;
 RESOURCE_KIND_POD = 6;
 RESOURCE_KIND_VOLUME = 7;
}

// RiskLevel represents the risk level of a Kubehound resource.
enum RiskLevel {
 RISK_LEVEL_UNSPECIFIED = 0;
 RISK_LEVEL_HIGH = 1;
 RISK_LEVEL_MEDIUM = 2;
 RISK_LEVEL_LOW = 3;
}

// AnalyzeRequest represents a request to analyse a Kubehound resource.
message AnalyzeRequest {
  // The kind of Kubehound resource.
  ResourceKind kind = 1;
  // The Kubehound resource will be analysed in JSON format.
  google.protobuf.Struct resource = 2;
}

// AnalyzeResponse represents the response of the risk analysis.
message AnalyzeResponse {
  // Whether the resource is compromised.
  bool is_compromised = 1;
  // The risk level of the resource.
  RiskLevel risk_level = 2;
}
```

</details>

### Risk Engine Embedding

The risk engine will be embedded in the core codebase as a separate module in a 
dedicated process. The forked process will provide a gRPC stream endpoint and 
the core codebase will be responsible for communicating with the risk engine and
for handling the results of the risk analysis.

The risk engine will be configurable using a configuration file handled by the 
core codebase. The configuration file will define the risk engine to use and the
arguments to pass to the executable or the gRPC risk engine endpoint address.

According to the risk engine complexity, the risk engine can be run as a separate
process managed by the codebase or as a unmanaged standalone service.

#### Standalone Executable

> `kubehound ingest` will start the risk engine as a separate process and will
> communicate with the risk engine using gRPC streams.

A standalone executable that is run as a separate process started by the 
`kubehound` command. The risk engine process will be managed by the core codebase
and bound to the command lifecycle.

Service configuration:

```yaml
# The configuration for the risk engine.
risk-engine:
  # The risk engine is running as a separate process.
  service:
    # The core codebase will execute the risk engine.
    managed:
      # The path to the risk engine executable.
      command: /usr/local/bin/risk-engine
      # The SHA-512 checksum of the risk engine executable.
      sha512: 1234567890abcdef
      # The arguments to pass to the risk engine executable.
      args: [
        "--listen-address=unix:///var/run/risk-engine.sock",
      ]
    # The risk engine is running as a separate process.
    client:
      # The address of the gRPC risk engine endpoint.
      address: unix:///var/run/risk-engine.sock
```

#### Risk Engine as Service

> `kubehound ingest` will communicate with the pre-existing risk engine using 
> gRPC streams.

An existing service that is run as a separate process called by the `kubehound`
command.

Using a pre-existing service that is run as an unmanaged standalone service is a
deployment option for the risk engine when acting as an independent system 
consuming additional dynamic data, not managed by the core codebase.

Service configuration:

```yaml
# The configuration for the risk engine.
risk-engine:
  # The risk engine is running as a separate service.
  service:
    client: 
      # The address of the gRPC risk engine endpoint.
      address: tcp://localhost:50051
```

### Evaluation

The proposed risk engine externalisation strategy has the following pros and cons:

| Evaluation | Comment |
|------------|---------|
| Pros       | - The risk engine is externalised as a separate module. <br> - The risk engine can be implemented in any language supporting gRPC. <br> - The checks can implement complex data-driven logic. <br> - Ingestion flexibility where resources could be altered during their processing. |
| Cons       | - The risk engine is a standalone artifact to deploy and manage. <br> - Introduce network calls during data ingestion that could slow down the whole process or introduce failures. |

# Threat Model

## Security Concerns

The risk engine will be responsible for analysing the risk of the Kubehound
resources. The output of the risk engine will be used to influence the graph
properties and to determine the critical paths in the graph. There is no direct
impact on the security of the cluster or the resources. At minima, the risk 
engine will return an incorrect risk analysis result which is consistent with a 
false positive or a false negative.

## Privacy Concerns

The risk engine is not responsible for storing or transmitting sensitive/personal
data. The risk engine will only analyse the Kubehound resources and return the
risk analysis results. No personal data should be processed intentionally.

# History

- 2024-12-23 - `draft-1`
  - Bootstrap the KEP
- 2025-01-02 - `draft-2`
  - Add configuration-based risk engine

# References

- [KubeHound](https://kubehound.io)
- [gRPC](https://grpc.io)
- [Streaming Data with gRPC](https://programmingpercy.tech/blog/streaming-data-with-grpc/)
