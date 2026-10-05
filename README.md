# VNet & Storage Configuration (Project 2 of 6)

Hands-on Azure lab built for AZ-104 (Microsoft Azure Administrator) preparation. It covers Azure networking and storage security: extending an existing virtual network with a dedicated subnet, deploying a storage account with public network access disabled, and connecting it privately through a private endpoint with full private DNS integration. The environment is defined as Infrastructure-as-Code in Bicep and builds directly on the VM/RBAC environment from Project 1.

> Related repos: [VM-RBAC-Config](https://github.com/dwaynec-cloud/VM-RBAC-Config) (Project 1) · [Monitoring-Backup-Config](https://github.com/dwaynec-cloud/Monitoring-Backup-Config) (Project 3) · [Entra-Identity-Config](https://github.com/dwaynec-cloud/Entra-Identity-Config) (Project 4) · [AppService-Config](https://github.com/dwaynec-cloud/AppService-Config) (Project 5) · [Storage-Recovery-Config](https://github.com/dwaynec-cloud/Storage-Recovery-Config) (Project 6)

> **Status:** the lab environment was torn down in October 2026 when the Azure free trial ended. This repo is kept as documentation of the build.

---

## Architecture

- **Virtual Network** (extended from Project 1): `vnet-vmrbac-project`, with a second subnet, `subnet-private-endpoints` (`172.16.1.0/24`), dedicated to private endpoints and kept separate from the VM's subnet
- **Network Security Group** (extended from Project 1): `nsg-vmrbac-project`, with explicit Deny rules limiting the VM's subnet to VNet-only traffic: no unsolicited inbound traffic from within the VNet and no outbound internet access (see the Project 3 note under Key decisions)
- **Storage Account**: `stvmrbacproject01`, created with public network access disabled
- **Private Endpoint**: `pe-storage-vmrbac`, connecting the storage account's blob service into `subnet-private-endpoints`
- **Private DNS Zone**: `privatelink.blob.core.windows.net`, linked to the VNet and attached to the private endpoint through a DNS zone group. This makes the storage account's normal hostname resolve to its private IP (`172.16.1.4`) from inside the VNet instead of to a public address
- **Infrastructure as Code**: the second subnet, storage account, NSG rules, private endpoint, and DNS resources are defined in `storage-network.bicep`, which references Project 1's VNet with the `existing` keyword rather than redeclaring it

<img width="612" height="432" alt="Project 2 architecture diagram" src="https://github.com/user-attachments/assets/65fd7cdd-25fb-4f60-a733-8461372f9887" />

The VM sits in its own subnet with internet access blocked, and the private endpoint sits in its dedicated subnet with a private IP. The solid arrow shows the private-link data path to the storage account. The dashed arrow shows the DNS zone resolving the storage account's hostname to that private IP, which is a name-resolution path rather than a data connection.

## Key decisions

**Extended the existing VNet rather than building a new one.** Project 2's storage and networking work is part of the same environment as Project 1's VM, so I added a second subnet to `vnet-vmrbac-project` instead of creating an isolated VNet. The Bicep template reflects this by referencing the VNet with the `existing` keyword. Real environments usually build incrementally on shared infrastructure, so this is closer to practice than a template that redeclares everything.

**Chose full VNet isolation over allowing outbound internet.** The project spec asked for explicit inbound and outbound restrictions. I chose the stricter option, denying all outbound internet and allowing only VNet-internal traffic, over a looser one that would also have allowed OS updates. The tradeoff is stronger isolation at the cost of the VM being unable to reach the internet at all, including its own package repositories.

> **Later change (Project 3):** this posture also blocked the Azure Monitor Agent. Project 3 added outbound allow rules for the `AzureMonitor`, `AzureResourceManager`, and `AzureActiveDirectory` service tags at a higher priority than the Deny rule. General internet access stayed blocked. See [Monitoring-Backup-Config](https://github.com/dwaynec-cloud/Monitoring-Backup-Config).

**The Portal and the CLI behaved differently for the same task, twice.**
- Creating the private endpoint in the Portal was blocked by Project 1's tag-enforcement policy, because the wizard couldn't tag the network interface it creates automatically for the endpoint. The same operation through Azure CLI, with the tag supplied, succeeded.
- The Portal wizard configures private DNS integration automatically. `az network private-endpoint create` does not. With the CLI, the Private DNS Zone, the VNet link, and the DNS zone group have to be created as separate steps. Until they existed, the storage account's hostname still resolved to a public address from inside the VNet.

## Challenges & troubleshooting

**`resourceGroup().location` returned a stale value.** This Bicep function returns the resource group's own location metadata, not where the resources inside it live. `rg-vmrbac-project` still recorded `canadaeast` from its original creation (before the VM was rebuilt in North Central US after repeated capacity failures), while every real resource was in `northcentralus`. Using the function for the private endpoint's location caused a deployment failure: the template tried to create a resource with a duplicate name in the wrong region. Fixed by using an explicit `location` parameter.

**`--what-if` caught an unintended change before deployment.** Running `az deployment group create --what-if` against the live environment showed the template would flip `privateEndpointNetworkPolicies` on the private-endpoint subnet from `Disabled` back to `Enabled`, overwriting a deliberate setting. Declaring the property explicitly in the template fixed it. This is the clearest example in the project of why `--what-if` should run before a deployment against a live environment.

**An output on an unchanged `existing` resource failed the deployment.** An output that read a live property (`customDnsConfigs`) from a resource the template didn't modify caused `DeploymentOutputEvaluationFailed`. Removing that output resolved it.

## Verification

- **Private DNS resolution:** from inside the VM, `nslookup stvmrbacproject01.blob.core.windows.net` resolves to `172.16.1.4`, the private endpoint's IP, rather than to a public address
- **Outbound internet blocked:** from inside the VM, `curl https://www.google.com` times out, confirming the NSG's outbound Deny rule is active
- **Public network access disabled:** set on the storage account, so the account is reachable only through the private endpoint from inside the VNet
- **Inbound restricted to SSH from one known IP:** unchanged from Project 1
- **Bicep template matches the live environment:** `--what-if` showed only a minor tag standardization as a real change, and the actual deployment completed with `provisioningState: Succeeded`

**Private DNS resolution**

<img width="863" height="613" alt="nslookup resolving the storage account to the private endpoint IP" src="https://github.com/user-attachments/assets/b2ef8a6d-2fec-4b92-9e48-25a18cc0e1ed" />

**Outbound internet blocked**

<img width="558" height="71" alt="curl to the internet timing out from inside the VM" src="https://github.com/user-attachments/assets/bb03a3cf-7475-4885-8479-79576625c811" />

## How to deploy

This template assumes Project 1's VNet already exists (see [VM-RBAC-Config](https://github.com/dwaynec-cloud/VM-RBAC-Config)).

```bash
git clone https://github.com/dwaynec-cloud/VNet-Storage-Config.git
cd VNet-Storage-Config
az login

# Preview changes first
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file storage-network.bicep \
  --what-if

# Deploy
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file storage-network.bicep
```

To connect to the VM and verify private connectivity:

```bash
ssh -i <path-to-your-key> azureuser@<vm-public-ip>
nslookup <your-storage-account>.blob.core.windows.net
```

## Next steps

- Compare this hand-written template with an Azure Verified Module (AVM) equivalent for the storage account
- Consider relaxing the outbound posture to allow only the package repositories the VM needs, so it can receive OS updates without general internet access
