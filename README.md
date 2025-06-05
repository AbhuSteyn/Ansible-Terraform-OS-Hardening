
# Terraform & Ansible Integration for Azure VM Provisioning and OS Hardening

This repository demonstrates an end-to-end workflow for provisioning Azure Virtual Machines with Terraform and then applying OS hardening using Ansible. Instead of using an inventory plugin, we generate the Ansible inventory dynamically using a Jinja2 template based on Terraform outputs. The solution also details how SSH connections are securely established using key-based authentication while following best practices for secrets management.

> **Important:**  
> - **Never commit private keys** (or any sensitive data) to version control.  
> - Use environment-managed secrets and tools like `ssh-copy-id` to securely populate your remote VM's authorized keys.
> - When running in CI/CD or automated environments, use a secrets manager or CI/CD secrets mechanism rather than storing keys in plain text.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Step 1: Provision Azure VMs with Terraform](#step-1-provision-azure-vms-with-terraform)
5. [Step 2: Generate Dynamic Inventory via Templating](#step-2-generate-dynamic-inventory-via-templating)
6. [Step 3: OS Hardening with Ansible](#step-3-os-hardening-with-ansible)
7. [SSH Connection Details in Ansible](#ssh-connection-details-in-ansible)
8. [Running the Workflow](#running-the-workflow)
9. [Best Practices](#best-practices)
10. [Additional Resources](#additional-resources)

---

## Overview

This solution is divided into three major stages:

1. **Provisioning Azure VMs with Terraform:**  
   Terraform creates the required Azure resources (resource group, virtual network, public IP, and a Linux VM) and outputs the VM public IP address.

2. **Dynamic Inventory Generation via Templating:**  
   A Jinja2 template is used to generate an Ansible hosts file from a JSON file containing Terraform outputs. This file is later used by Ansible as its inventory.

3. **OS Hardening with Ansible:**  
   Ansible connects over SSH (using secure key-based authentication) and applies OS hardening measures (e.g., updating packages, disabling root SSH login).

---

## Prerequisites

- **Terraform:** Version 0.12+  
- **Ansible:** Version 2.9+  
- **Python & Jinja2:** Python should be installed, along with the Jinja2 package if you plan to use the Python templating script.
- **Azure CLI (Optional):** For authentication and additional resource management.
- **SSH Key Pairs:** Create your SSH key pair locally. **DO NOT store the private key in GitHub.** Instead, manage it securely with environment variables, an SSH agent, or secrets management in your CI/CD system.

---

## Project Structure

A suggested directory layout for this solution is:

```
project/
├── terraform/
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
├── ansible/
│   ├── playbooks/
│   │   └── os_hardening.yml
│   ├── inventories/
│   │   ├── inventory_template.j2   # Jinja2 template for hosts file
│   │   └── hosts                   # Generated static inventory file
│   ├── vm_outputs.json             # (Will be created after terraform output)
│   └── ansible.cfg
├── scripts/
│   └── generate_inventory.py       # Python helper for templating
└── README.md
```

---

## Step 1: Provision Azure VMs with Terraform

Terraform is used to provision your Azure VMs and supporting resources. A simplified example is shown below.

**Example `terraform/main.tf`:**

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "rg-example"
  location = var.location
}

resource "azurerm_virtual_network" "example" {
  name                = "vnet-example"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "example" {
  name                 = "subnet-example"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "example" {
  name                = "pubip-example"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  allocation_method   = "Dynamic"
}

resource "azurerm_network_interface" "example" {
  name                = "nic-example"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.example.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.example.id
  }
}

resource "azurerm_linux_virtual_machine" "example" {
  name                = "vm-example"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  size                = var.vm_size
  admin_username      = var.admin_username

  network_interface_ids = [
    azurerm_network_interface.example.id,
  ]

  admin_ssh_key {
    username   = var.admin_username
    public_key = file(var.public_key_path)
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
}
```

**Example `terraform/outputs.tf`:**

```hcl
output "vm_public_ip" {
  description = "The public IP of the Azure VM"
  value       = azurerm_public_ip.example.ip_address
}
```

After running `terraform apply`, export the outputs to a JSON file for use by Ansible:

```bash
cd terraform
terraform output -json > ../ansible/vm_outputs.json
```

---

## Step 2: Generate Dynamic Inventory via Templating

Using the Terraform outputs, a Jinja2 template generates an Ansible hosts file.

**Example `ansible/inventories/inventory_template.j2`:**

```jinja
{% set vm_data = lookup('file', '../ansible/vm_outputs.json') | from_json %}
[azure_vms]
{{ vm_data.vm_public_ip.value }}
```

You can render this template using a Python script.

**Example `scripts/generate_inventory.py`:**

```python
#!/usr/bin/env python3
import json
from jinja2 import Environment, FileSystemLoader

# Load the Terraform outputs
with open('../ansible/vm_outputs.json') as f:
    data = json.load(f)

# Set up the Jinja2 environment for the inventories directory
env = Environment(loader=FileSystemLoader('../ansible/inventories'))
template = env.get_template('inventory_template.j2')

# Render the dynamic inventory with Terraform data
inventory_content = template.render(vm_public_ip=data["vm_public_ip"])

# Write the generated inventory to the hosts file
with open('../ansible/inventories/hosts', 'w') as f:
    f.write(inventory_content)
```

Run the script to generate the `hosts` file:

```bash
cd scripts
python3 generate_inventory.py
```

The generated `hosts` file (located in `ansible/inventories/hosts`) lists the Azure VM public IP under the `azure_vms` group and is used by Ansible for targeting hosts.

---

## Step 3: OS Hardening with Ansible

After generating the updated inventory, use Ansible to perform OS hardening on your Azure VMs.

**Example `ansible/playbooks/os_hardening.yml`:**

```yaml
- name: OS Hardening on Azure VM
  hosts: azure_vms
  become: yes
  tasks:
    - name: Update system packages (Ubuntu/Debian)
      apt:
        update_cache: yes
        upgrade: dist
      when: ansible_os_family == "Debian"

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: Restart SSH Service

  handlers:
    - name: Restart SSH Service
      service:
        name: ssh
        state: restarted
```

---

## SSH Connection Details in Ansible

### How It Works

Ansible connects to the Azure VM(s) over SSH using key-based authentication. Here’s how this is set up securely:

1. **SSH Key Pair Generation and Distribution:**

   - **Generate Your Key Pair Locally:**  
     Use the `ssh-keygen` utility on your local machine to generate your key pair.
     
     ```bash
     ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
     ```
     
     This creates a private key (e.g., `~/.ssh/id_rsa`) and a public key (e.g., `~/.ssh/id_rsa.pub`). **Never store the private key in your repository.**

   - **Distribute the Public Key:**  
     Use `ssh-copy-id` to copy your public key to the remote Azure VM. This method appends your public key to the `~/.ssh/authorized_keys` on the VM so that key-based login is permitted.
     
     ```bash
     ssh-copy-id -i ~/.ssh/id_rsa.pub azureuser@<vm_public_ip>
     ```
     
     This command ensures that the remote VM recognizes your public key without storing sensitive keys in GitHub.

2. **Using Secrets and Environment Variables:**

   - **CI/CD Pipelines and Secrets:**  
     When automating deployments in your CI/CD pipeline, do not hard-code your private key. Instead, store it as a secret in your CI/CD system or use a secure secrets manager. At runtime, the CI/CD pipeline can inject the private key into the environment.
     
   - **SSH Agent:**  
     Alternatively, add your private key to an SSH agent on your management system. The agent holds the key in memory, preventing the need to reference the key file directly in your configuration.
     
     ```bash
     eval "$(ssh-agent -s)"
     ssh-add ~/.ssh/id_rsa
     ```

3. **Ansible Configuration Without Hard-Coded Keys:**

   In the `ansible/ansible.cfg` file, avoid hardcoding the private key path if possible. If your environment supplies the key via an SSH agent or CI/CD secrets, simply specify the remote user. You may optionally reference a key file (ensuring that it’s kept secure and not checked into GitHub) during local testing.
   
   **Example `ansible/ansible.cfg`:**
   
   ```ini
   [defaults]
   # Use the generated inventory file
   inventory = inventories/hosts
   remote_user = azureuser
   # For local testing only; in production, rely on an SSH agent or secrets injection.
   private_key_file = ~/.ssh/my_azure_private_key.pem
   host_key_checking = False

   [ssh_connection]
   ssh_args = -o ControlMaster=auto -o ControlPersist=60s
   ```
   
   For production deployments, you can remove (or override) `private_key_file` if the SSH agent is properly configured.

With this setup, Ansible connects securely over SSH using the credentials installed on your Azure VM via `ssh-copy-id` and the secrets managed outside of your code repository.

---

## Running the Workflow

1. **Provision Azure VMs with Terraform:**

   ```bash
   cd terraform
   terraform init
   terraform apply -auto-approve
   ```

2. **Export Terraform Outputs:**

   ```bash
   terraform output -json > ../ansible/vm_outputs.json
   ```

3. **Generate the Ansible Inventory:**

   ```bash
   cd scripts
   python3 generate_inventory.py
   ```

4. **(Optional) Wait for VM Readiness:**  
   Allow enough time (or use a playbook with the `wait_for` module) to ensure that the remote VMs are accepting SSH connections.
   
   ```bash
   sleep 120  # Wait for 2 minutes for VM initialization
   ```

5. **Run the OS Hardening Playbook with Ansible:**

   ```bash
   cd ../ansible
   ansible-playbook playbooks/os_hardening.yml -vvvv
   ```

Ansible will connect via SSH using your secure key-based authentication (with keys distributed by `ssh-copy-id` or managed via environment secrets) and apply the OS hardening tasks.

---

## Best Practices

- **Secure Secrets:**  
  Never commit private keys or sensitive configuration to version control. Use SSH agents, environment variable–based secrets, or CI/CD secrets management.
  
- **Dynamic Inventory via Templating:**  
  Automatically generate your hosts file from Terraform outputs to avoid stale data.
  
- **Idempotence:**  
  Write your playbook tasks so that they can be re-run without causing unwanted changes.
  
- **Separation of Concerns:**  
  Keep the infrastructure provisioning (Terraform) separate from configuration management (Ansible).
  
- **Verbose Logging:**  
  Use increased verbosity (`-vvvv`) during troubleshooting.
  
- **Wait for VM Readiness:**  
  Ensure that VMs are fully booted and accepting SSH connections before running configuration tasks.

---

## Additional Resources

- **Terraform Documentation:** [Terraform by HashiCorp](https://www.terraform.io/docs)
- **Azure Provider for Terraform:** [AzureRM Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- **Ansible Documentation:** [Ansible Docs](https://docs.ansible.com)
- **Jinja2 Templating:** [Jinja2 Documentation](https://jinja.palletsprojects.com)
- **Ansible Vault:** [Ansible Vault Guide](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- **SSH Best Practices:**  
  Read about secure SSH key management and using `ssh-copy-id` from man pages and online resources.

---

This workflow demonstrates how to integrate Azure VM provisioning (via Terraform) with robust OS hardening (via Ansible) while ensuring secure SSH connectivity without exposing private keys in version control. With dynamic inventory templating and secrets managed outside the repository, you establish a scalable, secure automation pipeline.

Happy provisioning and hardening!
