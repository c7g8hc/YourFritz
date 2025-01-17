To achieve your goal of provisioning a Windows 10 VM with Packer, disabling the "WAP" network adapter, retrieving the IPv4 address of the "MGMT" network adapter, and setting the default gateway to `.1`, you can follow these steps. I'll break down the scripts into parts that you can copy into a PowerShell script and use them with Packer for automation.

### Step-by-step plan:
1. **Disable the "WAP" network adapter.**
2. **Retrieve the IPv4 address of the "MGMT" network adapter.**
3. **Set the default gateway of the "MGMT" adapter to `.1`.**

### PowerShell Script: `win10-setup.ps1`

```powershell
# Disable the 'WAP' network adapter (you can adjust the name if needed)
$WapAdapter = Get-NetAdapter | Where-Object {$_.Name -like '*WAP*'}
if ($WapAdapter) {
    Write-Host "Disabling WAP network adapter..."
    Disable-NetAdapter -Name $WapAdapter.Name -Confirm:$false
} else {
    Write-Host "WAP adapter not found. Skipping..."
}

# Get the 'MGMT' network adapter (you can adjust the name if needed)
$MgmtAdapter = Get-NetAdapter | Where-Object {$_.Name -like '*MGMT*'}
if ($MgmtAdapter) {
    Write-Host "Found MGMT network adapter: $($MgmtAdapter.Name)"

    # Get the IPv4 address of the MGMT adapter
    $IpAddress = (Get-NetIPAddress -InterfaceAlias $MgmtAdapter.Name -AddressFamily IPv4).IPAddress
    Write-Host "IPv4 Address of MGMT Adapter: $IpAddress"
    
    # Set the default gateway of the MGMT adapter to .1
    $Gateway = "$($IpAddress.Split('.')[0..2] -join '.').1"
    Write-Host "Setting default gateway to $Gateway"
    
    Set-NetIPAddress -InterfaceAlias $MgmtAdapter.Name -DefaultGateway $Gateway
} else {
    Write-Host "MGMT adapter not found. Skipping..."
}
```

### Explanation:
1. **Disable WAP Adapter:**
   - The script identifies the "WAP" network adapter using `Get-NetAdapter` and disables it with `Disable-NetAdapter` if it exists. Adjust the adapter name if necessary.
   
2. **Retrieve the MGMT Adapter's IPv4 Address:**
   - The script identifies the "MGMT" network adapter (adjust the name as needed) and retrieves its IPv4 address using `Get-NetIPAddress`.
   
3. **Set the Gateway to `.1`:**
   - The script calculates the gateway IP by taking the first three octets of the IPv4 address from the "MGMT" adapter and appending `.1` (e.g., if the IP is `192.168.1.10`, the gateway will be set to `192.168.1.1`).
   - It then sets the default gateway for the MGMT adapter using `Set-NetIPAddress`.

### Usage with Packer:
You can use this script in your Packer configuration file (JSON or HCL) as part of the provisioning step. For example, in a Packer template using the `windows-setup.ps1` file:

```json
{
  "builders": [
    {
      "type": "hyperv-iso",
      "iso_url": "path_to_windows10_iso",
      "iso_checksum": "checksum_value",
      "communicator": "winrm",
      "winrm_username": "Administrator",
      "winrm_password": "password",
      "vm_name": "win10-vm",
      "disk_size": 61440,
      "floppy_files": [
        "scripts/win10-setup.ps1"
      ]
    }
  ],
  "provisioners": [
    {
      "type": "powershell",
      "scripts": ["scripts/win10-setup.ps1"]
    }
  ]
}
```

Make sure to adjust the path to `win10-setup.ps1` according to your project structure.

### Notes:
- Ensure that the adapter names (`*WAP*` and `*MGMT*`) match the exact names of your network adapters on the Windows 10 VM.
- The `Set-NetIPAddress` command will modify the gateway, so it's necessary to have administrator rights.
- If you need to perform additional network configurations or handle different network interfaces, you might need to adjust the script accordingly.

Let me know if you need further adjustments or additional details!



Sure! Here’s how you can convert the Packer configuration to HCL2 format, which is the HashiCorp Configuration Language used in Packer's newer versions.

### Packer Configuration in HCL2

```hcl
# Define the builder to create a Windows 10 VM
source "hyperv-iso" "win10" {
  iso_url            = "path_to_windows10_iso"  # Provide the correct path to your Windows 10 ISO
  iso_checksum       = "checksum_value"         # Provide the checksum of the ISO
  communicator       = "winrm"
  winrm_username     = "Administrator"          # Windows administrator username
  winrm_password     = "password"               # Windows administrator password
  vm_name            = "win10-vm"
  disk_size          = 61440                     # Disk size in MB
  floppy_files       = ["scripts/win10-setup.ps1"] # Copy the PowerShell script to the VM during provisioning

  # Adjust any specific settings for your environment
  # e.g., network settings, number of CPUs, RAM, etc.
}

# Define the provisioner to run PowerShell script after the VM is created
build "win10" {
  provisioner "powershell" {
    scripts = ["scripts/win10-setup.ps1"]  # Path to your PowerShell script
  }
}
```

### Explanation of the HCL2 Configuration:

1. **Source Block (`source "hyperv-iso" "win10"`)**:
   - Specifies the source configuration for building a Windows 10 VM using the `hyperv-iso` builder.
   - Replace `"path_to_windows10_iso"` with the actual path to your Windows 10 ISO file.
   - Provide the ISO checksum for verification in the `"iso_checksum"` field.
   - Set the `winrm_username` and `winrm_password` to the appropriate credentials for the VM.
   - The `floppy_files` array includes the `win10-setup.ps1` PowerShell script that will be copied to the VM during the build process.

2. **Build Block (`build "win10"`)**:
   - Defines a `provisioner` block that will execute the `win10-setup.ps1` script after the VM is created.
   - This is where your PowerShell provisioning script will run, which includes disabling the "WAP" network adapter, retrieving the IPv4 address of the "MGMT" adapter, and setting the default gateway.

### Example Directory Structure

```
project_directory/
│
├── scripts/
│   └── win10-setup.ps1   # PowerShell script to configure the VM
│
└── packer.hcl             # Packer configuration file in HCL2 format
```

### How to Use the Configuration:
1. **Ensure the PowerShell Script Exists**: 
   Place the `win10-setup.ps1` script in the `scripts/` directory, as specified in the HCL2 configuration.

2. **Run Packer Build**: 
   Execute the following command in your terminal within the project directory:

   ```bash
   packer build packer.hcl
   ```

   This will use the `hyperv-iso` builder to create the Windows 10 VM and run the `win10-setup.ps1` script during provisioning.

----

To achieve the tasks you've described using Packer on Linux/macOS, we need to break it down into a series of steps. Below is an HCL (HashiCorp Configuration Language) script that covers the following:

1. **Clone a Windows 10 template** using the `vsphere` Packer builder.
2. **Run a PowerShell script** to elevate privileges and enable internet routing on the cloned VM.
3. **Run Windows Update** using Packer's `windows-update` plugin.
4. **Revert the network settings** (disable internet routing).
5. **Create a new template** from the updated image.

Here is the complete `packer.hcl` script to accomplish this:

```hcl
# Packer Configuration for cloning and updating a Windows 10 VM template in vSphere

# Required variables
variable "vsphere_server" {
  type    = string
  default = "vsphere.local"
}

variable "vsphere_user" {
  type    = string
  default = "your-username"
}

variable "vsphere_password" {
  type    = string
  default = "your-password"
}

variable "vsphere_datacenter" {
  type    = string
  default = "your-datacenter"
}

variable "vsphere_datastore" {
  type    = string
  default = "your-datastore"
}

variable "vsphere_cluster" {
  type    = string
  default = "your-cluster"
}

variable "vsphere_template_name" {
  type    = string
  default = "win10-template"
}

variable "vsphere_vm_name" {
  type    = string
  default = "win10-clone"
}

variable "vsphere_network" {
  type    = string
  default = "VM Network"
}

variable "vsphere_resource_pool" {
  type    = string
  default = "Resources"
}

# Packer Builder for vSphere
source "vsphere-iso" "windows10" {
  vcenter_server   = var.vsphere_server
  user             = var.vsphere_user
  password         = var.vsphere_password
  datacenter       = var.vsphere_datacenter
  datastore        = var.vsphere_datastore
  cluster          = var.vsphere_cluster
  network          = var.vsphere_network
  resource_pool   = var.vsphere_resource_pool
  template         = var.vsphere_template_name
  vm_name          = var.vsphere_vm_name
  wait_for_guest_ip_timeout = "30m"
  
  # Customize the Windows 10 VM settings if needed
  customize {
    windows_options {
      computer_name = "win10-clone"
    }
  }
}

# Provisioner to run PowerShell commands for network configuration (Internet routing)
build {
  sources = ["source.vsphere-iso.windows10"]

  provisioner "powershell" {
    inline = [
      # Elevate PowerShell to administrator
      "$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()",
      "$currentPrincipal = New-Object System.Security.Principal.WindowsPrincipal($currentUser)",
      "if (-not ($currentPrincipal.IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator))) {",
      "  $arguments = '-NoProfile -ExecutionPolicy Bypass -Command ""& {Start-Process powershell -ArgumentList \\"-NoProfile -ExecutionPolicy Bypass -Command \\'$MyInvocation.MyCommand.Definition\\'\\" -Verb RunAs}""'",
      "  Start-Process powershell -ArgumentList $arguments -Verb RunAs",
      "  exit",
      "}",
      
      # Enable the internet routing (modify network adapter)
      "Enable-NetAdapter -Name 'Ethernet0'",
      "New-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '192.168.1.1'",
      "Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('8.8.8.8', '8.8.4.4')"
    ]
  }

  provisioner "windows-update" {
    # Run Windows Update
    update_command = "wuauclt /detectnow /updatenow"
  }

  provisioner "powershell" {
    inline = [
      # Revert network settings to block internet route (disable internet routing)
      "Set-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '0.0.0.0'",
      "Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('0.0.0.0')"
    ]
  }

  provisioner "vsphere" {
    # Create new template from updated VM
    template_name = "win10-clone-updated-template"
    clone_name    = var.vsphere_vm_name
    wait_for_guest_ip_timeout = "30m"
  }
}

# Define the output of the built image
output "vsphere_vm_ip" {
  value = "${build.vsphere-iso.windows10.ip_address}"
}

```

### Explanation of the Script:

1. **Cloning the Windows 10 Template:**
   - The `vsphere-iso` builder is used to clone an existing Windows 10 VM template (`win10-template`), specifying the vSphere connection parameters, datacenter, datastore, network, and resource pool. The cloned VM will be named `win10-clone`.

2. **PowerShell Script to Enable Internet Routing:**
   - The first PowerShell provisioner ensures that the script is run with administrative privileges (using `Start-Process` to re-launch PowerShell as an administrator).
   - It then configures the network adapter (`Ethernet0`) to enable internet access by assigning a static IP, gateway, and DNS servers.

3. **Windows Update:**
   - The `windows-update` provisioner uses `wuauclt` to force Windows to detect and apply available updates.

4. **Revert Network Changes:**
   - Another PowerShell provisioner is used to disable internet routing by removing the default gateway and DNS settings on the network adapter.

5. **Create New Template:**
   - The final step uses the `vsphere` provisioner to create a new template (`win10-clone-updated-template`) from the updated VM.

### Notes:
- You will need to ensure that Packer is properly configured to communicate with your vSphere environment, including having the necessary credentials and access to the vSphere API.
- The network adapter name (`Ethernet0`) is assumed, but you may need to adjust it depending on your Windows image.
- This script is tailored to clone and update a Windows 10 template. Modify variables such as IP addresses and DNS settings based on your specific network environment.


# ---
You're right. The error you encountered is because `vsphere` is not a provisioner in Packer. The `vsphere` functionality is part of the **builder** configuration, not a provisioner. So, the attempt to use `vsphere` as a provisioner will result in an error.

To fix this, we need to correct the final step of creating a template from the updated VM. We cannot use a `vsphere` provisioner for that. Instead, we'll use the `vsphere-iso` builder to handle the cloning, updating, and creating the new template.

### Corrected Packer HCL Script

```hcl
# Packer Configuration for cloning and updating a Windows 10 VM template in vSphere

# Required variables
variable "vsphere_server" {
  type    = string
  default = "vsphere.local"
}

variable "vsphere_user" {
  type    = string
  default = "your-username"
}

variable "vsphere_password" {
  type    = string
  default = "your-password"
}

variable "vsphere_datacenter" {
  type    = string
  default = "your-datacenter"
}

variable "vsphere_datastore" {
  type    = string
  default = "your-datastore"
}

variable "vsphere_cluster" {
  type    = string
  default = "your-cluster"
}

variable "vsphere_template_name" {
  type    = string
  default = "win10-template"
}

variable "vsphere_vm_name" {
  type    = string
  default = "win10-clone"
}

variable "vsphere_network" {
  type    = string
  default = "VM Network"
}

variable "vsphere_resource_pool" {
  type    = string
  default = "Resources"
}

# Packer Builder for vSphere
source "vsphere-iso" "windows10" {
  vcenter_server   = var.vsphere_server
  user             = var.vsphere_user
  password         = var.vsphere_password
  datacenter       = var.vsphere_datacenter
  datastore        = var.vsphere_datastore
  cluster          = var.vsphere_cluster
  network          = var.vsphere_network
  resource_pool   = var.vsphere_resource_pool
  template         = var.vsphere_template_name
  vm_name          = var.vsphere_vm_name
  wait_for_guest_ip_timeout = "30m"
  
  # Customize the Windows 10 VM settings if needed
  customize {
    windows_options {
      computer_name = "win10-clone"
    }
  }

  # Revert network adapter to block internet access (in case of a reboot)
  post_shutdown_delay = "10s"
}

# Provisioner to run PowerShell commands for network configuration (Internet routing)
build {
  sources = ["source.vsphere-iso.windows10"]

  provisioner "powershell" {
    inline = [
      # Elevate PowerShell to administrator
      "$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()",
      "$currentPrincipal = New-Object System.Security.Principal.WindowsPrincipal($currentUser)",
      "if (-not ($currentPrincipal.IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator))) {",
      "  $arguments = '-NoProfile -ExecutionPolicy Bypass -Command ""& {Start-Process powershell -ArgumentList \\"-NoProfile -ExecutionPolicy Bypass -Command \\'$MyInvocation.MyCommand.Definition\\'\\" -Verb RunAs}""'",
      "  Start-Process powershell -ArgumentList $arguments -Verb RunAs",
      "  exit",
      "}",
      
      # Enable the internet routing (modify network adapter)
      "Enable-NetAdapter -Name 'Ethernet0'",
      "New-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '192.168.1.1'",
      "Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('8.8.8.8', '8.8.4.4')"
    ]
  }

  provisioner "windows-update" {
    # Run Windows Update
    update_command = "wuauclt /detectnow /updatenow"
  }

  provisioner "powershell" {
    inline = [
      # Revert network settings to block internet route (disable internet routing)
      "Set-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '0.0.0.0'",
      "Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('0.0.0.0')"
    ]
  }

  # Once the VM is updated, create a new template from the updated VM.
  # This is done by using the "vsphere-iso" builder and setting the `template` flag.

  # New template creation is handled by builder itself, no separate provisioner required
  post_process {
    inline = [
      "export VM_ID=$(govc vm.info -json -k / -u ${var.vsphere_user}:${var.vsphere_password}@${var.vsphere_server} ${var.vsphere_vm_name} | jq -r '.VirtualMachines[0].MoRef.Value')",
      "govc vm.markastemplate -k / -u ${var.vsphere_user}:${var.vsphere_password}@${var.vsphere_server} -vm $VM_ID"
    ]
  }
}

# Define the output of the built image
output "vsphere_vm_ip" {
  value = "${build.vsphere-iso.windows10.ip_address}"
}
```

### Key Updates:

1. **Create a Template from Updated VM**:
   - The `vsphere` provisioner was removed because it's not a valid provisioner in Packer. Instead, the creation of a new template is handled by a **post-processing step** using `govc` (a command-line utility for interacting with VMware vSphere environments).
   - `govc` is used to mark the newly cloned and updated VM as a template by calling the `govc vm.markastemplate` command.

2. **Post-processing Step**:
   - The `post_process` block uses `govc` to interact with vSphere and mark the VM as a template after the VM has been successfully updated. This requires the `govc` tool to be installed and available in the environment where you're running Packer.
   - The `govc` tool interacts with the vSphere server using environment variables or command-line arguments to authenticate and perform actions like converting the VM into a template.

### Prerequisites:

- **`govc` tool**: You will need to install the `govc` command-line utility to perform the post-processing step. It's part of the `govmomi` project and can be found [here](https://github.com/vmware/govmomi/tree/master/govc).
  - Ensure that `govc` is accessible in your environment.
  - You can install it via `brew install govmomi` on macOS or download it from GitHub on Linux.

### How it Works:
- The `vsphere-iso` builder clones the template, runs the network configuration, applies Windows updates, and then reverts the network settings.
- The `post_process` block uses `govc` to mark the updated VM as a template in vSphere.

This approach avoids the error with the `vsphere` provisioner and leverages post-processing for creating the new template.


```powershell
# setup.ps1

# Elevate PowerShell to administrator if not already running as administrator
$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$currentPrincipal = New-Object System.Security.Principal.WindowsPrincipal($currentUser)

if (-not ($currentPrincipal.IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator))) {
    $arguments = '-NoProfile -ExecutionPolicy Bypass -Command ""& {Start-Process powershell -ArgumentList \\"-NoProfile -ExecutionPolicy Bypass -Command \\'$MyInvocation.MyCommand.Definition\\'\\" -Verb RunAs}""'
    Start-Process powershell -ArgumentList $arguments -Verb RunAs
    exit
}

# Enable the internet routing (modify network adapter)
Write-Host "Enabling network adapter for internet routing..."
Enable-NetAdapter -Name 'Ethernet0'
New-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '192.168.1.1'
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('8.8.8.8', '8.8.4.4')

# Run Windows Update
Write-Host "Running Windows Update..."
Start-Process "cmd.exe" -ArgumentList "/c wuauclt /detectnow /updatenow" -NoNewWindow -Wait

# Revert network changes (disable internet routing)
Write-Host "Reverting network adapter settings..."
Set-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.1.100' -PrefixLength 24 -DefaultGateway '0.0.0.0'
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses ('0.0.0.0')

Write-Host "Setup completed successfully!"
```
