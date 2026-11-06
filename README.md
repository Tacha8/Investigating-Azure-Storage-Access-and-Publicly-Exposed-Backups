# Investigating Azure Storage Access and Publicly Exposed Backups

## Scenario

A developer needed to understand how an application accesses customer data stored in Azure Blob Storage. The investigation traced the application's storage account, authentication configuration, managed identity permissions, private endpoint, and Key Vault secret. It also examined a separate backup storage account to determine whether its data was exposed through anonymous access.

## Environment

- **Cloud platform:** Microsoft Azure
- **Resource scope:** Isolated Azure training environment
- **Storage:** Azure Blob Storage (StorageV2)
- **Identity provider:** Microsoft Entra ID
- **Identity type:** User-assigned or system-assigned managed identity (as configured in the lab)
- **Authorization:** Azure RBAC — Storage Blob Data Contributor
- **Secrets management:** Azure Key Vault
- **Network security:** Azure Private Endpoint and private DNS
- **Investigation tools:** Azure Portal, web browser, XML container listing
- **Access level:** Reader on the lab resource group; Key Vault Secrets User scoped to the lab Key Vault
- **Changes performed:** Read-only investigation; no resource configuration changes

## Investigation

### 1. Identifying the application's storage account

I located the primary storage account, `stmadhatlabchap6`, and inspected its Blob service endpoint.

- Identified the Blob endpoint: `https://stmadhatlabchap6.blob.core.windows.net/`.
- Confirmed the account uses StorageV2 (general purpose v2) with Standard performance.
- Verified that secure transfer is required.
- Confirmed the minimum TLS version is 1.2.

The Blob endpoint identifies the service hostname the application can use to address the storage account.

**Evidence — Primary Blob service endpoint**

<img width="1191" height="453" alt="image" src="https://github.com/user-attachments/assets/445ce0e6-65f9-4c10-bf1a-727723054d62" />


### 2. Examining storage authentication settings

I inspected the storage account's Configuration settings to understand which authentication methods were permitted.

Key findings:

- **Allow storage account key access:** Disabled.
- **Allow Blob anonymous access:** Disabled.
- **Secure transfer required:** Enabled.
- **Minimum TLS version:** 1.2.

Disabling Shared Key authorization prevents clients from authenticating to this storage account using account keys. Disabling anonymous Blob access also prevents unauthenticated public access through the account's Blob configuration.

These settings indicate that the application needs an alternative supported authorization method, such as Microsoft Entra ID.

**Evidence — Storage account security configuration**

<img width="547" height="617" alt="image" src="https://github.com/user-attachments/assets/62169da6-30dc-44b1-81f9-3b49a422dcdc" />


### 3. Identifying the application's managed identity permissions

I inspected the storage account's Access Control (IAM) and reviewed the managed identity's current role assignments.

Findings:

- Identified the application identity as `id-lab-app-chap6`.
- Found one role assignment: `Storage Blob Data Contributor`.
- The assignment was scoped directly to the primary storage account.
- No deny assignments were displayed in the inspected view.

The Storage Blob Data Contributor role provides read, write, and delete permissions over Blob data.

This means the application identity has the ability to read and modify Blob contents, rather than being restricted to read-only access.

**Security observation:** If the application only needs to upload or retrieve specific customer records, a narrower role or more limited scope could reduce the impact of a compromised application identity.

**Evidence — Managed identity role assignment**

<img width="782" height="532" alt="image" src="https://github.com/user-attachments/assets/3fd6a5c7-225b-4ffc-aa80-adc97aba793f" />


### 4. Tracing the private endpoint and DNS configuration

I located the private endpoint named `pe-lab-storage` and inspected its DNS configuration.

Findings:

- Private endpoint IP address: `10.70.1.36`.
- Blob service FQDN: `stmadhatlabchap6.blob.core.windows.net`.
- The endpoint's DNS configuration showed the Blob hostname mapped to `10.70.1.36`.

This demonstrates the intended private connectivity path: the application can use the standard Blob service hostname while DNS directs traffic to the private endpoint's IP address.

**Evidence limitation:** The DNS configuration view showed the hostname and private IP mapping, but the DNS zone group table displayed no results. The storage account's public network access setting and the private endpoint connection status were not independently captured in the screenshots.

**Evidence — Private endpoint DNS configuration**

<img width="1397" height="611" alt="image" src="https://github.com/user-attachments/assets/220f93eb-faae-4b7f-bd2f-70e3e44c6dfb" />


## Investigation

### 5. Locating the application's Key Vault secret

I navigated to the Azure Key Vault and located the application's connection secret.

Findings:

- Identified the secret `app-db-connection`.
- Confirmed that its current version was enabled.
- The secret is intended to provide the application's storage connection configuration.

The intended application access flow is:

1. The application authenticates using its managed identity.
2. Microsoft Entra ID provides an identity token.
3. The application uses its Key Vault permissions to retrieve the connection secret.
4. The application uses the retrieved configuration to connect to its storage service.

**Authentication consideration:** The primary storage account has Shared Key authorization disabled. Therefore, a conventional connection string containing an account key would not be compatible with that setting. The actual secret authentication method must be inspected to determine whether it uses Microsoft Entra ID authentication or another supported approach.

The secret value is intentionally excluded from this portfolio because connection strings may contain credentials and other sensitive configuration.

**Evidence limitation:** The Key Vault screenshot confirms that the secret exists and is enabled. It does not independently prove successful retrieval by the application or establish the authentication method contained in the secret.

**Evidence — Key Vault secret version**

<img width="1498" height="327" alt="image" src="https://github.com/user-attachments/assets/be3de5a0-0c6e-476f-8e93-d4a146679118" />


### 6. Investigating the backup storage account

I investigated a separate Azure storage account and its backup container to determine whether stored data was accessible without authentication.

I accessed the container listing endpoint through a browser without supplying authentication credentials:

`https://<backup-storage-account>.blob.core.windows.net/db-backups?restype=container&comp=list`

The endpoint returned an XML `EnumerationResults` response containing the container's contents.

Findings:

- Identified the `db-backups` container.
- Successfully retrieved the container listing without authentication.
- Discovered a blob named `customers-export-2026.json`.
- Content type: `application/json`.
- Content length: 822 bytes.
- Last modified: July 15, 2026, 02:26:41 GMT.

The response exposed the blob's name, URL, size, content type, and modification timestamp without authentication.

**Security finding:** The backup container permits anonymous enumeration. An unauthenticated party can discover the names and metadata of objects stored in the container.

**Evidence limitation:** The XML response confirms anonymous container listing. It does not independently establish that the JSON file's contents can also be downloaded anonymously. Content readability requires separate validation.

**Evidence — Anonymous Blob container listing**

<img width="983" height="612" alt="image" src="https://github.com/user-attachments/assets/cfaa3e12-3512-4bf8-b2bc-7441805fd620" />


## What broke / what surprised me

- The primary storage account had Shared Key authorization disabled. This meant I could not assume that a conventional account-key connection string would work. The connection string's authentication method needs to be checked against the storage account's authorization configuration.
- The managed identity had Storage Blob Data Contributor rather than a read-only role. This was broader access than an application limited to reading customer data might require.
- The private endpoint DNS configuration showed the Blob hostname mapped to a private IP, but the DNS zone group table displayed no results. This reinforced the importance of distinguishing a DNS mapping shown in a configuration view from a fully validated DNS integration.
- The most significant discovery was the separate backup account. While the primary account's configuration disabled anonymous Blob access and Shared Key authorization, the backup container returned its contents listing without authentication.
- The public listing exposed a customer-export filename. Even without proving that the file itself was readable, exposing backup object names and metadata is a security concern.

## Findings and recommendations

### Findings

1. The primary storage account had Shared Key authorization and anonymous Blob access disabled.
2. The application managed identity had Storage Blob Data Contributor permissions at the storage account scope.
3. The private endpoint DNS configuration mapped the primary Blob hostname to private IP `10.70.1.36`.
4. An enabled Key Vault secret containing the application's connection configuration was present.
5. A separate backup storage account allowed anonymous listing of its `db-backups` container, exposing the name and metadata of a customer export.

### Recommendations

**1. Restrict anonymous access to backup storage**

Disable anonymous Blob access at the storage account level and ensure containers do not permit public access. Review the backup account's public network access configuration and restrict connectivity to approved private networks where practical.

**2. Apply least privilege to the application identity**

Review whether the application genuinely requires read, write, and delete permissions. If its workload only requires a subset of those operations, use a more restrictive role or narrower scope where supported.

**3. Validate the complete application authentication and network flow**

Inspect the connection configuration to determine its authentication method and confirm compatibility with Shared Key authorization being disabled. Validate managed identity access to Key Vault, private DNS resolution, private endpoint connectivity, and the storage account's public network access settings.

## What I learned

- **Identity and data-plane permissions:** Storage Blob Data Contributor grants permissions over Blob contents, which is different from having management permissions over the storage account resource.
- **Shared Key versus Microsoft Entra ID:** Disabling account key access changes which authentication methods clients can use. A connection string should not automatically be assumed to use a compatible authentication method.
- **Private endpoints and DNS:** Private connectivity depends on more than creating a private endpoint. DNS must resolve the service hostname to the intended private IP, and the network path must permit connectivity.
- **Key Vault's role:** Key Vault protects application secrets, but storing a secret securely does not guarantee that the application uses it safely or that the target resource's configuration is compatible.
- **Investigating exposure:** Anonymous container listing and anonymous Blob downloading are separate capabilities. A security investigation should validate each one before declaring the contents publicly exposed.
