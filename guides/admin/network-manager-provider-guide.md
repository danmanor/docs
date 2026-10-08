# OSAC Networking Manager Provider Guide

This guide explains how a cloud provider implements and registers Fabric and
Kubernetes Managers for OSAC networking. It follows the provider workflow from
choosing a role through registering data models, wiring managers together, and
checking the resulting behavior.

> **Implementation status:** This guide describes the target contract in the
> [Network Manager Integration Contract Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md).
> The current OSAC code still uses the legacy ConfigMap-based manager registry
> and does not yet enforce the generic NetworkDataModel, NetworkManager, and
> NetworkData contract. The generic OSAC platform support described below must
> ship before a provider can use this workflow end to end.

## Contents

- [How manager integration works](#how-manager-integration-works)
- [Before you start](#before-you-start)
- [Step 1: Choose the manager role](#step-1-choose-the-manager-role)
- [Step 2: Define any data exchanged between managers](#step-2-define-any-data-exchanged-between-managers)
- [Step 3: Implement the AAP manager role](#step-3-implement-the-aap-manager-role)
- [Step 4: Register models, managers, and NetworkClass](#step-4-register-models-managers-and-networkclass)
- [Step 5: Verify the integration](#step-5-verify-the-integration)
- [Changing or removing a manager](#changing-or-removing-a-manager)
- [Implementation status and conformance ownership](#implementation-status-and-conformance-ownership)
- [Related documents](#related-documents)

## How manager integration works

OSAC gives tenants one networking API across virtual machines, managed
Kubernetes clusters, and bare-metal servers. The same `VirtualNetwork`,
`Subnet`, and other networking resources keep the same meaning across those
workloads. A provider supplies backend implementations behind that API:

- The **Fabric Manager** configures the provider's physical network and handles
  shared networking resources and physical workload interfaces. Every
  `NetworkClass` must select one.
- The **Kubernetes Manager** configures Kubernetes-side networking for VM
  overlays. It is optional when VM workloads are not supported. It does not
  replace the Fabric Manager.

The managers are independent. They do not call one another or receive one
another's credentials. If a Kubernetes Manager needs a value produced by the
Fabric Manager, OSAC carries that value between the jobs as `NetworkData`.
`NetworkDataModel` defines the value's name, owner, and JSON Schema;
`NetworkData` stores one validated value for that model and owner. A
`NetworkManager` registers a role implementation and declares its input or
output model names. A `NetworkClass` selects the managers by their logical
names.

The registration check compares exact model names: every model declared in a
Kubernetes Manager's `networkInputs` must also be declared in the selected
Fabric Manager's `networkOutputs`. This catches missing data dependencies
before the `NetworkClass` is stored. It does not prove that the managers
implement the same network behavior; providers must verify that separately.

## Before you start

1. Confirm that the OSAC release includes the generic manager support described
   in the implementation status note. That support includes the registration
   APIs and CRDs, fulfillment-service validation, NetworkData schema
   validation, manager dispatch, and owner-scoped data lifecycle.
2. Decide which workload services the deployment will offer. Fabric Manager
   support is required. Select a Kubernetes Manager if the deployment offers
   VMaaS; without one, OSAC rejects ComputeInstance requests. CaaS and BMaaS
   still use the Fabric Manager.
3. Prepare the provider's Ansible Automation Platform (AAP) execution
   environment (EE) and Ansible collection. The fully qualified collection role named by
   `NetworkManager.spec.implementationRef` must be installed in the EE used by
   OSAC.
4. Identify the manager's non-secret backend settings and the AAP credentials
   it needs. Register settings under `NetworkManager.spec.configuration` and
   reference AAP Credential IDs under `spec.aapCredentialIds`; OSAC passes only
   that manager's settings and credential references to its job. Keep secret
   values in AAP Credential objects.

The fulfillment-service database is authoritative for provider registrations.
The Kubernetes custom resources in the networking hub are projections watched
by the operator. Submit registrations through the provider-facing
fulfillment-service API; do not create or edit the projected CRDs with
`kubectl`.

## Step 1: Choose the manager role

### Fabric Manager

The Fabric Manager handles shared networking resources and physical workload
interfaces. Its required operation set includes creating and deleting
VirtualNetworks and Subnets, applying and removing policy on Cluster and
BaremetalInstance interfaces, managing ExternalIP pools and allocations,
managing ExternalIP attachments and NAT gateways, and querying Dynamic Host
Configuration Protocol (DHCP) leases for Clusters and BaremetalInstances. If
Ethernet east-west support is enabled, it also creates and deletes
FabricDomains.

### Kubernetes Manager

The Kubernetes Manager handles Kubernetes-side Subnet networking and applies
or removes network attachment policy for ComputeInstance VM overlays. It does
not handle shared VirtualNetwork operations, physical workload interfaces, or
Fabric Manager work. A Kubernetes Manager is optional for deployments without
VM workloads.

The normative resource meanings and tenant-visible behavior are in the
[Unified Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-unified-networking/design.md).
In particular, implementations must preserve these shared semantics:

- A `VirtualNetwork` is a Layer 3 (L3) routing domain. Subnets in it can route
  to each other, subject to SecurityGroup policy.
- Each `Subnet` is a separate Layer 2 (L2) broadcast domain. Workloads attached to the
  same Subnet share that L2 domain.
- SecurityGroup rules are stateful, apply to the workload's network attachment,
  combine as a union, allow established reply traffic, and deny unmatched new
  traffic.
- The current shared networking contract is IPv4-only and permits one tenant
  network attachment per workload.
- ExternalIP attachment provides inbound translation; NATGateway provides
  outbound source translation. They have separate lifecycles and behavior.

Capabilities are a predefined OSAC list, not provider-defined strings. The
only capability in this contract is `EAST_WEST_ETHERNET`, serialized in the
manager API as `NETWORK_MANAGER_CAPABILITY_EAST_WEST_ETHERNET`. The Fabric
`NetworkManager.spec.capabilities` field declares that an implementation can
run Ethernet east-west operations. The deployment enables that behavior by
setting `NetworkClass.spec.east_west_capabilities.supports_east_west_ethernet`
to true. Both declarations are required: fulfillment-service rejects a
NetworkClass that enables the behavior when its selected Fabric Manager lacks
the capability; when the NetworkClass field is false or omitted, OSAC rejects
a FabricDomain request before AAP dispatch. A capability on a manager alone
does not enable tenant requests.

When enabled, `fabric_domain.create` and `fabric_domain.delete` use the same
generic Fabric Manager registration, collection role, AAP template,
configuration, and credential binding as the other operations. Backend
settings such as a provider template name go in that manager's
`spec.configuration`; NetworkClass contains only the deployment-level
capability toggle. OSAC passes configuration through without interpreting
provider keys. The Fabric Manager validates its required settings and
preserves the FabricDomain, VirtualNetwork, and Subnet semantics defined in
the [Unified Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-unified-networking/design.md)
and [Multi-Fabric East-West Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1382-multi-fabric-east-west-networking/design.md).

All networking resources, including `FabricDomain`, use the selected generic
manager contract; no resource has a separate vendor-specific dispatch path.

## Step 2: Define any data exchanged between managers

Create a `NetworkDataModel` only for a value that needs a durable owner-scoped
lifecycle or must pass from one manager role to another. Do not put values,
credentials, or secrets in the model itself. A model's name, meaning, owner
scope, and schema are immutable. If any of them must change, register a new
model name.

For each model, decide:

1. **Name and meaning:** Use a provider-owned DNS label such as
   `acme-networking-subnet-vlan-id`. Names beginning with `osac-` are reserved
   for OSAC models. Equal JSON types do not make two values interchangeable;
   the name is part of the contract.
2. **Owner scope:** Choose one supported owner scope: `NetworkClass`,
   `VirtualNetwork`, or `Subnet`. The owner determines how long the value lives
   and which operations can receive it. A new owner kind requires OSAC platform
   support.
3. **Value schema:** Supply an inline JSON Schema Draft 2020-12 schema. The
   fulfillment service validates the schema when the model is created and
   validates every value against it. OSAC does not fetch schemas from URLs.
4. **Consumer:** Document which manager reads the value and what the manager
   must do with it. Put the model name in the Fabric Manager's outputs and in
   the Kubernetes Manager's inputs when it crosses between those roles.

For example, a VLAN-based Fabric Manager can publish one VLAN ID per Subnet,
which a Kubernetes `LocalNet`-style Manager can consume:

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkDataModel
metadata:
  name: acme-networking-subnet-vlan-id
spec:
  description: VLAN ID used to connect this Subnet to the provider fabric.
  ownerScope: Subnet
  schema:
    $schema: https://json-schema.org/draft/2020-12/schema
    type: integer
    minimum: 1
    maximum: 4094
```

For REST Create, send the resource fields without `apiVersion` and `kind`. For
example, this is a direct JSON request body for the same model:

```json
{
  "metadata": {"name": "acme-networking-subnet-vlan-id"},
  "spec": {
    "description": "VLAN ID used to connect this Subnet to the provider fabric.",
    "ownerScope": "Subnet",
    "schema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "integer",
      "minimum": 1,
      "maximum": 4094
    }
  }
}
```

Save that object as `network-data-model.json` and create it before either
manager registration:

```sh
curl --fail-with-body --request POST \
  "${FULFILLMENT_API}/api/private/v1/network_data_models" \
  --header "Authorization: Bearer ${OSAC_PROVIDER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data-binary @network-data-model.json
```

The schema is the model's value contract. For example, this schema permits an
integer from 1 through 4094; it does not itself configure a switch or create a
Kubernetes network. The Fabric Manager allocates and configures the VLAN, then
returns its value. The Kubernetes Manager consumes the validated value and
maps it to its own backend configuration.

Provider models may use JSON scalars, arrays, or objects. Use one composite
object when related fields share a meaning, owner, and lifetime. Use separate
models when values have different meanings, owners, or lifetimes. JSON Schema
checks structure and constraints; each manager remains responsible for the
resource-specific meaning described by the model.

## Step 3: Implement the AAP manager role

Implement the fully qualified Ansible collection role named by
`implementationRef` (the fully qualified collection name, or FQCN). The role
must cover **every** operation and workload
target assigned to its manager role. Providers cannot register only a subset.
The manager integration design names the task entry point for each operation:

| Operation | Manager role and target | Named task entry point |
|---|---|---|
| `virtual_network.create` / `virtual_network.delete` | Fabric; no workload target | `create_virtual_network` / `delete_virtual_network` |
| `subnet.create` / `subnet.delete` | Fabric and, when configured, Kubernetes; no workload target | `create_subnet` / `delete_subnet` |
| `fabric_domain.create` / `fabric_domain.delete` | Fabric, only when the manager declares `EAST_WEST_ETHERNET` and NetworkClass enables it | `create_fabric_domain` / `delete_fabric_domain` |
| `workload_attachment.apply` / `workload_attachment.delete` | Kubernetes for `ComputeInstance`; Fabric for `Cluster` and `BaremetalInstance` | `apply_workload_attachment` / `delete_workload_attachment` |
| `external_ip_pool.create` / `external_ip_pool.delete` | Fabric; no workload target | `create_external_ip_pool` / `delete_external_ip_pool` |
| `external_ip.allocate` / `external_ip.release` | Fabric; no workload target | `create_external_ip` / `delete_external_ip` |
| `external_ip_attachment.create` / `external_ip_attachment.delete` | Fabric; target may be `ComputeInstance`, `Cluster`, or `BaremetalInstance` | `attach_external_ip` / `detach_external_ip` |
| `nat_gateway.create` / `nat_gateway.delete` | Fabric; no workload target | `create_nat_gateway` / `delete_nat_gateway` |
| `dhcp_lease.query` | Fabric; target may be `Cluster` or `BaremetalInstance` | `query_dhcp_lease` |

A collection role stores each named task entry point as a task file. For
example, FQCN `acme.networking.fabric_manager` resolves to
`ansible_collections/acme/networking/roles/fabric_manager/`, with
`tasks/create_subnet.yml`, `tasks/delete_subnet.yml`, and the other Fabric
entry points under `tasks/`. The generic OSAC playbook dynamically includes
the registered FQCN and the fixed task file for the operation:

```yaml
- name: Invoke the selected network manager
  ansible.builtin.include_role:
    name: "{{ osac_job_vars.manager.implementationRef }}"
    tasks_from: "{{ osac_network_task_from }}"
```

OSAC computes `osac_network_task_from` from the operation table; the manager
cannot choose a different operation. The collection must be installed in the
AAP execution environment used by the generic OSAC job template. See the
[Ansible `include_role` documentation](https://docs.ansible.com/projects/ansible-core/2.17/collections/ansible/builtin/include_role_module.html)
and [collection role FQCN documentation](https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_using_playbooks.html).

Create AAP Credential objects with the credential types and injected fields
the provider role expects, and record their numeric IDs for the manager
registration. The generic AAP job template must allow credentials at launch
and must not carry a shared provider-backend credential. OSAC includes the
selected manager's `configuration` in `osac_job_vars` and passes only that
manager's `aapCredentialIds` in the AAP launch request's `credentials` field.
AAP injects secret values according to each credential's type; OSAC does not
copy them into job variables, NetworkData, or result artifacts. The job
identity must be authorized to use each referenced credential. Verify these
bindings before selecting the manager in a production NetworkClass.

A Fabric Manager that declares `EAST_WEST_ETHERNET` must implement both
FabricDomain entry points. Each task receives the FabricDomain, its associated
VirtualNetwork context, that manager's configuration, and applicable
NetworkData. Providers cannot register a partial operation set.

Create ordering for a Subnet is Fabric first, then Kubernetes when selected.
Delete ordering is Kubernetes first, then Fabric. For `workload_attachment`,
OSAC routes the operation to the manager that owns that interface: the
Kubernetes Manager for a VM overlay, and the Fabric Manager for Cluster worker
and bare-metal interfaces.

### Input to each operation

OSAC passes an `osac_job_vars` object to the AAP task. Its common fields are:

| Field | Meaning |
|---|---|
| `operation` | The fixed OSAC operation, such as `subnet.create`. |
| `manager` | The selected manager's logical `name`, normalized `role`, `implementationRef`, and non-secret `configuration`. |
| `resource` | The OSAC resource being reconciled, including its API version, kind, metadata, and spec. The [Unified Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-unified-networking/design.md) defines each resource's fields and meaning. |
| `network_data` | Existing values that apply to this operation. Kubernetes receives only declared, applicable inputs. Fabric receives existing values for reuse and cleanup. |
| `network_output_models` | For Fabric create/reconcile operations, the model and owner keys for values OSAC expects the Fabric task to produce. It is empty or absent when there are no requested outputs. |
| `workload_attachment` | For attachment operations, the normalized Subnet, interface, and complete SecurityGroup policy. |

`workload_attachment` is a sibling of `resource`. OSAC keeps `resource` as the
owning workload so its UID and generation identify the job. The normalized
attachment includes the resolved Subnet and parent VirtualNetwork, the
interface or interfaces, and the complete ingress and egress rules of each
attached SecurityGroup. On apply, establish the Subnet connection and install
policy before traffic is enabled. On delete, remove workload-owned policy and
detach the interface before workload teardown.

For a VM, the Kubernetes Manager receives an overlay interface; for Cluster
and BaremetalInstance targets, the Fabric Manager receives resolved physical
interfaces. The input shape and policy requirements are:

```yaml
operation: workload_attachment.apply
resource:
  kind: ComputeInstance
  metadata: {uid: "<workload UID>", generation: 3}
workload_attachment:
  subnet:
    uid: "<subnet UID>"
    name: app-subnet
    ipv4_cidr: 192.0.2.0/24
    virtual_network: {uid: "<network UID>", name: app-network}
  interfaces:
    - {type: overlay, name: primary}
  security_groups:
    - uid: "<group UID>"
      name: web
      ingress:
        - {protocol: TCP, port_from: 443, port_to: 443, ipv4_cidr: 0.0.0.0/0}
      egress: []
```

The [manager contract's workload attachment input](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md#workload-attachment-policy-input)
defines the physical-interface variants and all rule constraints.

A Fabric task may return only model names OSAC requested and owner kinds and
UIDs included in the operation context. Each `(model name, owner kind, owner
UID)` appears at most once. OSAC validates the full result and each value
before writing any NetworkData record. It must return every requested model
and owner key exactly once; an omitted, duplicate, extra, or invalid value
fails the complete result. A consuming Kubernetes task starts only after
those records have been persisted. Manager tasks must not write NetworkData
directly or call the fulfillment-service API. For example, a Fabric task that
allocates VLAN 120 for a Subnet returns the value inside the common result
envelope:

```yaml
osac_result:
  operation: subnet.create
  resourceUID: "<subnet UID>"
  observedGeneration: 3
  data:
    network_data:
      - model_name: acme-networking-subnet-vlan-id
        owner:
          kind: Subnet
          uid: "<subnet UID>"
        value: 120
```

The fulfillment service validates the result and creates a separate
`NetworkData` API object with a generated name. That object is OSAC's durable
record; the provider manager returns only the model name, owner, and value.
OSAC then passes the validated record to a Kubernetes Manager that declares
the model as an input:

```yaml
osac_job_vars:
  network_data:
    - model_name: acme-networking-subnet-vlan-id
      resource_kind: Subnet
      resource_uid: "<subnet UID>"
      value: 120
```

The consumer reads the provider-defined fields and maps their documented
meaning to its backend API. The model name identifies the contract; matching
JSON types alone do not make two values interchangeable.

### Return values, errors, and retries

Every successful operation, including `dhcp_lease.query`, returns one AAP
artifact named `osac_result`. The Ansible task publishes it with
`ansible.builtin.set_stats`; see the [Ansible `set_stats` documentation](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/set_stats_module.html).
The artifact includes the dispatched operation,
the owning resource UID, and the observed generation. Every result contains
`data.network_data`; use an empty list when the operation publishes no
NetworkData values. Add only the operation-specific field required by that
operation: `data.external_ip` for `external_ip.allocate` or
`data.dhcp_leases` for `dhcp_lease.query`.
If backend work fails, fail the Ansible task with a diagnostic; do not publish
a success result for incomplete work.

For an operation with no NetworkData output:

```yaml
- name: Return operation result to OSAC
  ansible.builtin.set_stats:
    data:
      osac_result:
        operation: subnet.create
        resourceUID: "<resource UID>"
        observedGeneration: 3
        data:
          network_data: []
```

For ExternalIP allocation, return the reserved address. The manager does not
patch the ExternalIP or any Kubernetes object:

```yaml
osac_result:
  operation: external_ip.allocate
  resourceUID: "<external IP UID>"
  observedGeneration: 1
  data:
    network_data: []
    external_ip:
      address: 198.51.100.19
```

OSAC reads the result from the exact tracked AAP job, validates its operation,
UID, and generation, then validates that the address is canonical IPv4 and is
inside the selected pool. OSAC writes the confirmed address to
`ExternalIP.status.address` through its existing status feedback path. A
missing or invalid result leaves the ExternalIP non-ready. Retrying the same
ExternalIP UID must return the same provider reservation.

DHCP lease lookup uses the same result envelope:

```yaml
osac_result:
  operation: dhcp_lease.query
  resourceUID: "<workload UID>"
  observedGeneration: 1
  data:
    network_data: []
    dhcp_leases:
      - subnet_ref: "<subnet UID>"
        interface: primary
        ip_address: 192.0.2.10
        mac_address: "02:00:00:00:00:10"
```

Return one unambiguous lease for each requested attachment. A missing or
ambiguous match fails the operation. A malformed, missing, stale, or mismatched
result fails the operation and does not make the resource Ready.

Create/apply and delete tasks must be idempotent by resource UID;
workload-attachment operations are also scoped by interface. A retry may be a
new AAP job with a new job ID. OSAC polls the exact job ID it launched and does
not treat another job's result as success. For Subnet create retries, reuse
NetworkData values already persisted by OSAC rather than allocating
replacements. During delete, keep values until consumer and Fabric cleanup have
succeeded.

## Step 4: Register models, managers, and NetworkClass

Create objects in dependency order through the provider-facing
fulfillment-service APIs:

1. Create every provider-defined `NetworkDataModel`.
2. Create the Fabric `NetworkManager` and, if needed, the Kubernetes
   `NetworkManager`.
3. Create the `NetworkClass` that selects those logical manager names.

The examples below show request objects sent to the provider-facing private
Fulfillment API; they are not `kubectl apply` manifests. For API requests,
send only resource fields such as `metadata` and `spec`; omit the CRD-only
`apiVersion` and `kind` fields shown in projected-object examples. The gRPC
package is `osac.private.v1`. REST Create sends the resource object as the request body;
the response returns the created object. The API generates each resource `id`.
References use `metadata.name`; Get and Delete use the generated `id`.

| Resource | gRPC service | Create | List | Get | Delete |
|---|---|---|---|---|---|
| NetworkDataModel | `NetworkDataModels` | `POST /api/private/v1/network_data_models` | `GET /api/private/v1/network_data_models` | `GET /api/private/v1/network_data_models/{id}` | `DELETE /api/private/v1/network_data_models/{id}` |
| NetworkManager | `NetworkManagers` | `POST /api/private/v1/network_managers` | `GET /api/private/v1/network_managers` | `GET /api/private/v1/network_managers/{id}` | `DELETE /api/private/v1/network_managers/{id}` |
| NetworkData | `NetworkData` | `POST /api/private/v1/network_data` (OSAC identity only) | `GET /api/private/v1/network_data` | `GET /api/private/v1/network_data/{id}` | `DELETE /api/private/v1/network_data/{id}` (OSAC identity only) |

Create the deployment's profile after those registrations with the existing
`NetworkClasses.Create` gRPC method or `POST /api/private/v1/network_classes`.
NetworkClass Create resolves the selected manager names and validates their
roles, capabilities, and declared data
compatibility before it stores the profile.

List accepts the private API CEL `filter` parameter. To find one registration
by its object name, use `this.metadata.name == "acme-fabric-manager"`.
Update and Patch are not supported. Cloud Infrastructure Admins use the
existing private API authentication for registrations; only OSAC service
identities create or delete runtime NetworkData. No new CLI is defined.
In REST JSON and generated gRPC clients, `NetworkManager.role` and
`NetworkManager.capabilities` use the protobuf enum values
`NETWORK_MANAGER_ROLE_FABRIC`, `NETWORK_MANAGER_ROLE_KUBERNETES`, and
`NETWORK_MANAGER_CAPABILITY_EAST_WEST_ETHERNET` shown in the examples.

To list registrations by name over REST, URL-encode the private API CEL
expression in the `filter` query parameter:

```sh
curl --fail-with-body --get \
  "${FULFILLMENT_API}/api/private/v1/network_managers" \
  --header "Authorization: Bearer ${OSAC_PROVIDER_TOKEN}" \
  --data-urlencode 'filter=this.metadata.name == "acme-fabric-manager"'
```

The List response contains the generated `id`; save it for later Get or
Delete. References in a NetworkClass or another registration continue to use
the stable resource name.

OSAC installs the common networking models. Use those model names for existing
OSAC-defined values; add a provider-defined model only for a distinct value
that is not already covered by the common models.

Example Fabric registration:

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkManager
metadata:
  name: acme-fabric-manager
spec:
  managerName: acme_fabric
  role: NETWORK_MANAGER_ROLE_FABRIC
  implementationRef: acme.networking.fabric_manager
  description: Fabric networking for the Acme provider
  capabilities:
    - NETWORK_MANAGER_CAPABILITY_EAST_WEST_ETHERNET
  configuration:
    controller_url: https://fabric-api.example.test
    ethernet_east_west:
      server_cluster_template: spectrum-x-gpu-template
  aapCredentialIds: [42]
  networkOutputs:
    - acme-networking-subnet-vlan-id
```

Example Kubernetes registration that consumes the same model:

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkManager
metadata:
  name: acme-kubernetes-manager
spec:
  managerName: acme_kubernetes
  role: NETWORK_MANAGER_ROLE_KUBERNETES
  implementationRef: acme.networking.kubernetes_manager
  description: Kubernetes VM networking for the Acme provider
  networkInputs:
    - acme-networking-subnet-vlan-id
```

The REST body for the Fabric registration is a JSON resource object with the
same fields. For example, save this object as `network-manager.json` for the
Create call above:

```json
{
  "metadata": {"name": "acme-fabric-manager"},
  "spec": {
    "managerName": "acme_fabric",
    "role": "NETWORK_MANAGER_ROLE_FABRIC",
    "implementationRef": "acme.networking.fabric_manager",
    "description": "Fabric networking for the Acme provider",
    "capabilities": ["NETWORK_MANAGER_CAPABILITY_EAST_WEST_ETHERNET"],
    "configuration": {
      "controller_url": "https://fabric-api.example.test",
      "ethernet_east_west": {"server_cluster_template": "spectrum-x-gpu-template"}
    },
    "aapCredentialIds": [42],
    "networkOutputs": ["acme-networking-subnet-vlan-id"]
  }
}
```

Create the manager with the JSON body, using the provider's existing private
API bearer token:

```sh
curl --fail-with-body --request POST \
  "${FULFILLMENT_API}/api/private/v1/network_managers" \
  --header "Authorization: Bearer ${OSAC_PROVIDER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data-binary @network-manager.json
```

The API's configured provider identity determines authorization; the request
body cannot grant itself permission.

`metadata.name` identifies the registration object. `spec.managerName` is the
logical manager name referenced by `NetworkClass`. `implementationRef` is the
Ansible collection role FQCN. Fabric Managers declare `networkOutputs` and
leave `networkInputs` empty; Kubernetes Managers declare `networkInputs` and
leave `networkOutputs` empty. Empty lists are valid. Every listed model name
must already exist.

Example `NetworkClass` body, shown as an illustrative fulfillment-service
object rather than a `kubectl` manifest:

```json
{
  "id": "acme-default",
  "metadata": {"name": "acme-default"},
  "fabricManager": "acme_fabric",
  "k8sManager": "acme_kubernetes",
  "spec": {
    "eastWestCapabilities": {"supportsEastWestEthernet": true}
  }
}
```

The `NetworkClass` selects the deployment's Fabric and optional Kubernetes
Manager. OSAC accepts the selection only if the names resolve to registrations
of the correct role and every Kubernetes input model is among the selected
Fabric Manager's output models. To enable Ethernet east-west support, its
specification also sets `east_west_capabilities.supports_east_west_ethernet`
to true; OSAC accepts that only if the selected Fabric Manager declares
`EAST_WEST_ETHERNET`. Provider template names and other backend settings remain
in that manager's `configuration`, never in NetworkClass. This is data-contract validation; it is not a
manager-name allowlist and it is not a behavioral conformance test. The
registration, model, and `NetworkClass` specs are immutable; changes require
replacing the object after removing its dependents. The fulfillment-service
API is authoritative, and the operator may wait for their hub CRD projections
before dispatching provider jobs.

The provider registration APIs support Create, Get, List, and Delete for
`NetworkDataModel` and `NetworkManager`. Update and Patch are rejected. OSAC
validates dependencies at the point where they are created:

| Create request | OSAC validation | Result on failure |
|---|---|---|
| `NetworkDataModel` | Supported owner scope, unique name, and valid inline JSON Schema | Reject the object and identify the invalid field or schema. |
| `NetworkManager` | Valid role-specific fields, unique `(role, managerName)`, and existing model references | Reject the object and identify the invalid declaration or model name. |
| `NetworkClass` | Each logical manager resolves to the expected role; all Kubernetes input model names are present in Fabric outputs | Reject before persisting the profile; no dependent manager job starts. |
| Fabric manager result | Correct operation, resource UID, generation, declared output names, operation-context owners, and schema-valid values | Reject the complete result; do not persist invalid NetworkData or start the consumer job. |
| Kubernetes manager dispatch | Every declared input applicable to the operation has one valid owner-scoped NetworkData value | Fail before dispatch if an input is absent or invalid. |

If a registration has committed but its hub projection is not ready yet, OSAC
waits and retries reconciliation without dispatching an AAP job. This prevents
an incomplete projected registry from being treated as a valid manager
selection.

## Step 5: Verify the integration

The provider must show that the implementation preserves the shared API
behavior and implements all assigned operations. Registration success alone
proves only that the declared objects and model references passed validation.

Use the acceptance checks below and the cases in the manager contract's
[Test Plan](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-network-manager-integration-contract-networking/testplan.md)
as the provider conformance suite. Run them against the AAP execution
environment and actual backend that will be selected. OSAC may use these cases
for its own supported implementations; it does not run a hosted provider
certification service.

### Registration and data exchange

- Create a schema-valid model, then confirm an invalid schema is rejected.
- Confirm a manager with a missing model reference is rejected.
- Confirm a `NetworkClass` with the wrong manager role or unresolved logical
  manager name is rejected.
- Confirm a `NetworkClass` is rejected when a Kubernetes input model is not
  declared by the selected Fabric Manager.
- Confirm a NetworkClass that enables Ethernet east-west is rejected unless
  the selected Fabric Manager declares `EAST_WEST_ETHERNET`; confirm a
  declaration on the manager alone does not enable FabricDomain requests.
- Provision a Subnet; confirm the Fabric task returns every requested value,
  OSAC stores one NetworkData value per model and owner, and the Kubernetes
  task receives the same validated value.
- Confirm an invalid or duplicate result, wrong owner UID, or schema-invalid
  value fails before the Kubernetes stage starts.
- Delete the Subnet; confirm the Kubernetes Manager and then Fabric Manager
  receive the existing values until cleanup finishes.

### Shared networking behavior

- Confirm Subnets within one VirtualNetwork have distinct L2 broadcast domains
  and L3 routing between them, subject to the attached SecurityGroups.
- Confirm workloads attached to one Subnet share its L2 broadcast domain.
- Confirm separate VirtualNetworks remain isolated.
- When Ethernet east-west is enabled, create and delete a `FabricDomain`
  through the selected Fabric Manager and confirm it does not change
  VirtualNetwork routing or Subnet broadcast domains.
- Confirm SecurityGroup rules apply on each network attachment, are stateful,
  combine as a union, allow established replies, and deny unmatched new
  traffic.
- Confirm policy is in effect before a workload is made Ready and is removed
  before workload teardown. A policy failure must leave traffic unavailable
  and the workload non-ready.
- Exercise these behaviors for each supported target: ComputeInstance VM
  overlay, Cluster worker interface, and BaremetalInstance physical interface.

### Resource operations and retry behavior

- Exercise every operation and workload-target row in [Step 3](#step-3-implement-the-aap-manager-role),
  including create and delete order for Subnets.
- Confirm create/apply and delete are safe to repeat for the same resource UID
  and that a failed attempt can be retried with a new AAP job ID.
- Confirm ExternalIP addresses are unique within a pool, a retry for one
  ExternalIP UID returns its same reservation, and release removes that
  reservation. Confirm the manager returns the address in
  `osac_result.data.external_ip.address`, OSAC validates it, and OSAC writes
  `ExternalIP.status.address` without manager access to the Kubernetes API.
- Confirm ExternalIP attachments are removed before an address is released,
  and NATGateway performs outbound SNAT without providing inbound access.
- Confirm DHCP lease lookup returns one unambiguous lease per requested
  attachment in `osac_result.data.dhcp_leases`, with the common operation,
  resource UID, generation, and `data.network_data` list present.

Do not declare a new implementation ready for tenant use until provider
conformance evidence covers every applicable operation, target, data model,
and shared resource behavior.

## Changing or removing a manager

`NetworkManager`, `NetworkDataModel`, and `NetworkClass` specs cannot be updated
or patched. To change an implementation reference, role, input/output list,
model meaning, owner scope, schema, or manager selection, register replacement
objects. The fulfillment service blocks deletion while dependencies remain:

- A `NetworkManager` cannot be deleted while a `NetworkClass` selects its role
  and logical manager name.
- A `NetworkDataModel` cannot be deleted while a manager or NetworkData value
  references it.
- NetworkData remains available until all manager cleanup operations that can
  consume it have succeeded.

Remove dependent tenant resources and the `NetworkClass` as required by the
Unified Networking lifecycle. Then delete registrations in reverse dependency
order and create their replacements. Do not assume that removing a manager
registration also removes provider-side network state.

## Implementation status and conformance ownership

This guide specifies the target provider contract: registration objects and
REST/gRPC operations, AAP role invocation and task entry points, per-manager
configuration and credential isolation, NetworkData exchange, operation
results, FabricDomain handling, ExternalIP allocation, and the minimum
behavioral acceptance checks. The guide describes a proposed OSAC contract;
the current OSAC code still uses the legacy ConfigMap-based manager registry
and does not yet enforce generic NetworkDataModel, NetworkManager, or
NetworkData support. Generic OSAC platform implementation must ship before
this onboarding flow can be used end to end.

The provider is responsible for verifying each implementation against the
acceptance checks in Step 5 and retaining its own test output and backend
observations as conformance evidence. Registration validation checks declared
models, roles, capabilities, and input/output compatibility; it cannot prove
backend behavior. OSAC does not operate a hosted provider certification
service. A deployment must not select a manager
for tenant use until its provider has verified every operation and applicable
target, including FabricDomain when Ethernet east-west is enabled.

## Related documents

- [Unified Networking PRD](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-unified-networking/prd.md)
- [Unified Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-unified-networking/design.md)
- [Network Manager Integration Contract PRD](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-network-manager-integration-contract-networking/prd.md)
- [Network Manager Integration Contract Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md)
- [Multi-Fabric East-West Networking Design](https://github.com/osac-project/enhancement-proposals/blob/main/enhancements/OSAC-1382-multi-fabric-east-west-networking/design.md)
