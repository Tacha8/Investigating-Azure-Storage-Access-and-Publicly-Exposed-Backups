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


### 5. Locating the application's Key Vault secret

I navigated to the `kv-madhat-lab-chap6` Key Vault and located the secret named `app-db-connection`.

Findings:

- Secret name: `app-db-connection`.
- Current version: `8f775041dccb4a46a90adb105375f0d`.
- Status: Enabled.

The secret's existence and enabled status establish that a connection configuration is stored in Key Vault.

The intended application access flow is:

1. The application authenticates using its managed identity.
2. Microsoft Entra ID provides an identity token.
3. The application uses its Key Vault permissions to retrieve the connection secret.
4. The application uses the retrieved configuration to connect to its storage service.

The actual secret value is intentionally excluded from this portfolio. Connection strings can contain credentials and other sensitive configuration.

**Evidence limitation:** The version listing confirms that the secret exists and is enabled. It does not independently prove successful retrieval by the application or
