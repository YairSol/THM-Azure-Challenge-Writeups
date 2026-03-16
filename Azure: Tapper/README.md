
## **Azure: Tapper - Cloud Pentest Writeup**

### 📝 **1. Introduction**

In this challenge, we were tasked with attacking an Azure tenant to discover an attack path and move laterally. While the "intended path" involved complex lateral movement through Entra ID (MFA bypass via TAP), this writeup demonstrates a more direct and creative approach: **Exploiting Azure Resource Manager (ARM) metadata** through the Cloud Shell.

---

### 🔍 **2. Initial Reconnaissance**

We started with a set of credentials and access to the Azure Portal via a low-privileged default user `<usr-XXXXX>`.

* **Target Resource:** A Virtual Machine named `VM1`.
* **Identified Users:** `goo`, `gumby`, and `pokey`.
* **The Obstacle:** An attempt to log in as the application owner (`gumby`) was blocked by **Multi-Factor Authentication (MFA)**, which effectively halted the intended web-portal attack path.

---

### 🚀 **3. Bypassing Entra ID via Azure Cloud Shell**

Instead of fighting the MFA at the **Identity** level, we shifted our focus to the **Infrastructure (Azure RBAC)** level. By using the **Azure Cloud Shell**, we leveraged the pre-authenticated environment of our current user to query the Azure Resource Manager (ARM) directly via CLI.

---

### 🛠️ **4. Exploiting Virtual Machine Metadata**

We discovered that our initial user had sufficient permissions to read the configuration settings of resources within the subscription. This allowed us to inspect the "Extensions" of the target Virtual Machine without needing to escalate privileges in Entra ID.

#### **Step 1: Resource Group Identification** 📂

First, we identified the specific Resource Group name for the lab session:

```bash
az group list --query "[].name" -o tsv

```

*Result: `<rg-XXXXX>*`

#### **Step 2: Inspecting VM Extensions** ⚙️

Azure VMs often use the **Custom Script Extension** to run provisioning scripts. These scripts and their parameters are stored in the VM's metadata. We ran the following command to retrieve this information:

```bash
az vm extension list --resource-group <rg-XXXXX> --vm-name VM1

```

---

### 🚩 **5. Flag Recovery (Exfiltration)**

The output of the command returned a JSON object for the `CustomScriptExtension`. Under the `settings` block, we found the cleartext command that was executed on the machine during its deployment.

**The Command Found in Metadata:**

```json
"settings": {
  "commandToExecute": "bash -c \"echo '<FLAG_VALUE>' > /tmp/flag.txt\""
}

```

By inspecting the **Control Plane** metadata, we successfully recovered the flag without needing to compromise additional user accounts or bypass MFA.

---

### 💡 **6. Lessons Learned & SOC Perspective**

* **Insecure Metadata:** Sensitive information like flags, passwords, or API keys should **never** be passed through VM extension parameters (`commandToExecute`), as they are readable by anyone with "Reader" permissions on the VM resource.
* **RBAC Over-Privilege:** This attack was possible because the initial low-privileged user had the right to "List" or "Read" VM extension settings—a common misconfiguration in large tenants.
* **Control Plane vs. Data Plane:** Security at the control plane (Azure Portal/CLI) is just as critical as security at the data plane (the OS inside the VM).

