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
