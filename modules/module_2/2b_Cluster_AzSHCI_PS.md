Module 2 - Scenario 2a - Clustering Azure Stack HCI with PowerShell
============
In this section, you'll walk through deployment of an Azure Stack HCI cluster using **PowerShell**. If you have a preference for deployment with Windows Admin Center, head over to the [Windows Admin Center cluster creation guidance](/modules/module_2/2a_Cluster_AzSHCI_WAC.md).

Contents <!-- omit in toc -->
-----------
- [Before you begin](#before-you-begin)
- [Step 1 - Management Tools](#step-1---management-tools)
- [Step 2 - Initial OS Configuration](#step-2---initial-os-configuration)
- [Step 3 - Install required features](#step-3---install-required-features)
- [Step 4 - Configuring networking](#step-4---configuring-networking)
  - [Shared compute and management with separate storage](#shared-compute-and-management-with-separate-storage)
  - [Shared compute and storage with separate management](#shared-compute-and-storage-with-separate-management)
  - [Shared compute and storage with separate management](#shared-compute-and-storage-with-separate-management-1)

Before you begin
-----------
At this stage, you should have completed the previous section of the workshop, [Deploying the Azure Stack HCI Infrastructure](/modules/module_2/2_Deploy_AzSHCI.md) and you should have a set of virtual machines running in your environment, visible in Hyper-V Manager:

![Workshop machines running](/modules/module_0/media/mslab_vms_running.png "Workshop machines running")

If you don't have those VMs running, go over and do that now - it should take about 10 minutes.

Azure Stack HCI cluster creation can be fully automated using PowerShell. In this scenario, 2a, we'll walk you through the steps required to configure an Azure Stack HCI cluster.

Step 1 - Management Tools
-----------
In this step, you'll install some additional management tools on your **HybridWorkshop-DC** virtual machine, that will assist in the remote configuration of the Azure Stack HCI environment.

1. On your Hyper-V host, open **Hyper-V Manager**.
2. Once open, you'll see your virtual machines up and running. Right-click on **HybridWorkshop-DC** and click **Connect**

![Connect to HybridWorkshop-DC](/modules/module_0/media/mslab_connect_dc.png "Connect to HybridWorkshop-DC")

3. In the **Connect to HybridWorkshop-DC** popup, use the **slider** to select your resolution and click **Connect**
4. When prompted, enter your **credentials** you provided in the **LabConfig** file. If you kept the default credentials, they will be:

    * **Username**: LabAdmin
    * **Password**: LS1setup!

5. Once logged into the Domain Controller VM, open **PowerShell as Administrator**.
6. Run the following code in your PowerShell window, and keep the window open for the next task.

```powershell
Install-WindowsFeature -Name RSAT-Clustering, RSAT-Clustering-Mgmt, `
    RSAT-Clustering-PowerShell, RSAT-Hyper-V-Tools, `
    RSAT-Feature-Tools-BitLocker-BdeAducExt, RSAT-Storage-Replica
```

This code essentially installs a number of **Remote Server Administration Tools (RSAT)** that allows more control of those specific roles and features, from this particular server. It should take a few minutes to install and configure.

Step 2 - Initial OS Configuration
---------
When running in production, there are certain tweaks and optimizations that can be made that enhance performance and reliability of the operating system. 2 examples of those optimizations include adjusting the **Memory Dump** settings, along with configuring the OS to operate in **High Performance** mode.

> If you're not familiar with an **Active Memory Dump**, it is particularly useful when your system is hosting virtual machines (VMs). When taking a regular complete memory dump, the contents of each VM is included. When there are multiple VMs running, this can account for a large amount of memory in use on the host system. Many times, the memory/issues of interest are in the parent host OS, not the child VMs. An active memory dump filters out the memory associated with all of the child VMs, ensuring the dump file sizes are more space efficient, and specific to the Hyper-V host.

In your **open PowerShell console**, run the code below to configure the **active memory dump settings**.

```powershell
# Define servers as variable
$Servers = "AzSHCI1", "AzSHCI2", "AzSHCI3", "AzSHCI4"

# Configure Active memory dump
Invoke-Command -ComputerName $servers -ScriptBlock {
    Set-ItemProperty -Path HKLM:\System\CurrentControlSet\Control\CrashControl -Name CrashDumpEnabled -value 1
    Set-ItemProperty -Path HKLM:\System\CurrentControlSet\Control\CrashControl -Name FilterPages -value 1
}

# Configure high performance power plan
# Set high performance if not VM
Invoke-Command -ComputerName $servers -ScriptBlock {
    if ((Get-ComputerInfo).CsSystemFamily -ne "Virtual Machine") {
        powercfg /SetActive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
    }
}
# Check settings
Invoke-Command -ComputerName $servers -ScriptBlock { powercfg /list }
```
_________________
**NOTE** - Configuring a high-performance power plan for a virtual machine will have no effect. It is just shown above for reference.
_________________

Step 3 - Install required features
-------
With the Azure Stack HCI OS ready, you can now install the additional features on each node, prior to configuring the Azure Stack HCI cluster. The required roles and features that need to be installed on your Azure Stack HCI nodes include:

* Hyper-V and Hyper-V PowerShell
* Failover Clustering
* Storage Replica and Storage Replica Management Tools
* BitLocker and BitLocker Management Tools
* Data Deduplication
* System Insights (Predictive analytics) and Management Tools

To perform the installation, still in your **administrative PowerShell console**, run the code below to install the necessary roles and features on your Azure Stack HCI nodes. 

```powershell
# Install Hyper-V using DISM if Install-WindowsFeature fails
# If nested virtualization is not enabled, Install-WindowsFeature fails
Invoke-Command -ComputerName $servers -ScriptBlock {
    $Result = Install-WindowsFeature -Name "Hyper-V" -ErrorAction SilentlyContinue
    if ($result.ExitCode -eq "failed") {
        Enable-WindowsOptionalFeature -FeatureName Microsoft-Hyper-V -Online -NoRestart 
    }
}

# Define and install features
$features = "Failover-Clustering", "Hyper-V-PowerShell", "Bitlocker", "RSAT-Feature-Tools-BitLocker", "Storage-Replica", "RSAT-Storage-Replica", "FS-Data-Deduplication", "System-Insights", "RSAT-System-Insights"
Invoke-Command -ComputerName $servers -ScriptBlock { Install-WindowsFeature -Name $using:features }

# Restart and wait for computers
Restart-Computer $servers -Protocol WSMan -Wait -For PowerShell -Force
Start-Sleep 20 # Allow time for reboots to complete fully
Foreach ($Server in $Servers) {
    do { $Test = Test-NetConnection -ComputerName $Server -CommonTCPPort WINRM }while ($test.TcpTestSucceeded -eq $False)
}
```
![Required features being installed on Azure Stack HCI](/modules/module_2/media/ps_install_features.png "Required features being installed on Azure Stack HCI")

Step 4 - Configuring networking
-------
With the roles and features successfully installed, and the nodes back online after a short reboot, it's time to configure the networking. Now, by default, your Azure Stack HCI nodes have 4 network adapters (NICs), however many new production systems are shipping with just 2 NICs, albeit with significantly higher performance and bandwidth available - in some cases, each NIC is 100GbE!

With as few as 2 physical NICs in a host, how can you ensure that you have multiple "separate networks" for traffic types like **Management**, **Virtual Machines** and **Storage**?

Fortunately, Azure Stack HCI provides a broad number of choices when it comes to defining the network infrastructure. Here's a few examples:

### Shared compute and management with separate storage
In the below example, the Azure Stack HCI node has 4 physical NICs (pNICS) - 

![Network diagram with shared compute and management with separate storage](/modules/module_2/media/network-atc-6-disaggregated-management-compute.png "Network diagram with shared compute and management with separate storage")

2 of the NICs are aggregated into a **Switch Embedded Team (SET)** - a specialized type of vSwitch-enabled NIC team that exists on the Hyper-V host, that aggregates the physical NICs, and the teaming/vSwitch functionality.

From there, a single virtual network adapter (vNIC) is created for the Host, that will reside on the designated management network, for example, the 192.168.0.0/24 network. Virtual Machines will also connect to this Team, and their traffic will also flow out via the teamed pNICs.

The remaining 2 pNICs are dedicated to storage traffic between the Azure Stack HCI nodes in the cluster. These don't need teaming, as the redundancy and performance across both adapters is provided by SMB Multichannel.

### Shared compute and storage with separate management
In the below example, again, the Azure Stack HCI node has 4 physical NICs (pNICS) - 

![Network diagram with compute and storage with separate management](/modules/module_2/media/network-atc-3-separate-management-compute-storage.png "Network diagram with compute and storage with separate management")

This example is particularly common when pNIC01 and pNIC02 are the onboard 1GbE adapters, which are most suited to management traffic, and the remaining 2 pNICs are high-performance adapters, useful for storage traffic, and the traffic coming in and leaving your virtual machines.

In this case, the pNICs are aggregated into 2 separate SET switches, one for management and one for the shared compute and storage, and corresponding host vNICs are created to allow the isolation of those different traffic types. Virtual machines would connect to the "Compute and Storage Team" in this example, sharing the total bandwidth provided by pNIC03 and pNIC04.

### Shared compute and storage with separate management
In the below example, the Azure Stack HCI node has just 2 physical NICs (pNICS) - 

![Network diagram with a fully converged network configuration](/modules/module_2/media/network-atc-2-full-converge.png "Network diagram with a fully converged network configuration")

In this example, we just have 2 high-performance NICs, yet multiple different traffic types (Compute, Management, Storage) to account for. With this example, you simply aggregate the 2 pNICs into a SET switch, and from there, create the appropriate host vNICs to provide the networks for the different traffic types. All different traffic types share the available pNIC SET bandwidth, so it's important to factor in QoS and traffic management here.

Now that you understand more about the options for network configurations, how do you actually go about **applying** the network configurations?

There are 2 ways: **Manually**, or with the new **Network ATC**.

With the **manual** approach, you're configuring each layer of the network across each node in the cluster - this can be complex, there are a number of moving parts, and things are easy to misconfigure or overlook. Also, occasionally things can change over time, configurations drift, which leads to additional challenges, especially around troubleshooting.

**Network ATC** however, simplifies the deployment and network configuration management for Azure Stack HCI clusters. This provides an intent-based approach to host network deployment. By specifying one or more intents (management, compute, or storage) for a network adapter, you can automate the deployment of the intended configuration. For more information on Network ATC, including an overview and definitions, see the [Network ATC overview](https://docs.microsoft.com/en-us/azure-stack/hci/concepts/network-atc-overview).

> Unfortunately, **deploying Network ATC in virtual environments is not supported**. Several of the host networking properties it configures are not available in virtual machines, which will result in errors.

For the purpose of this guide therefore, we'll be configuring the networking settings manually with PowerShell, and the steps below are tailored for use inside nested virtual machines. We'll choose the **Shared compute and storage with separate management** example from above.

1. On **HybridWorkshop-DC**, still in your **administrative PowerShell console**, run the code below to create the first SET for management

```powershell
# Define servers as variable
$Servers="AzSHCI1","AzSHCI2","AzSHCI3","AzSHCI4"

# Define the vSwitch Name
$vSwitchName = "Management Team"
Invoke-Command -ComputerName $servers -ScriptBlock {
    # Get the first 2 pNIC adapters on the system
    $NetAdapters = Get-NetAdapter | Where-Object Status -eq Up | Sort-Object Name | Select-Object -First 2
    # Create the VM Switch from those 2 adapters
    New-VMSwitch -Name $using:vSwitchName -EnableEmbeddedTeaming $TRUE -NetAdapterName $NetAdapters.Name
}
```

2. That will take a few moments, but once that has completed, you can confirm that the SET switch has been created on each of your cluster nodes by running the following PowerShell command:

```powershell
# Flush DNS
ipconfig /flushdns
# Retrieve vSwitch/Team information
Get-VMSwitch -CimSession $Servers | Select-Object Name,ComputerName
```

![Management team created with PowerShell](/modules/module_2/media/ps_management_team.png "Management team created with PowerShell")

3. Next, you'll create the corresponding shared compute and storage SET. Still in your **administrative PowerShell console**, run the code below to create the second SET. In this case, we include the **-AllowManagementOS = $FALSE** to ensure that the host doesn't create a vNIC at this stage.

```powershell
# Define servers as variable
$Servers = "AzSHCI1", "AzSHCI2", "AzSHCI3", "AzSHCI4"

# Define the vSwitch Name
$vSwitchName = "ComputeStorage Team"
Invoke-Command -ComputerName $servers -ScriptBlock {
    # Get the first 2 pNIC adapters on the system
    $NetAdapters = Get-NetAdapter | Where-Object Status -eq Up | `
        Where-Object { $_.Name -like "*3*" -or $_.Name -like "*4*" } | Sort-Object Name
    # Create the VM Switch from those 2 adapters
    New-VMSwitch -Name $using:vSwitchName -EnableEmbeddedTeaming $TRUE -NetAdapterName `
    $NetAdapters.Name -AllowManagementOS $FALSE
}
```

4. That will take a few moments, but once that has completed, you can confirm that the SET switch has been created on each of your cluster nodes by running the following PowerShell command:

```powershell
# Flush DNS
ipconfig /flushdns
# Retrieve vSwitch/Team information
Get-VMSwitch -CimSession $Servers | Select-Object Name,ComputerName
```

![All network teams now created with PowerShell](/modules/module_2/media/ps_teams_complete.png "All network teams now created with PowerShell")

5. Next, you'll **rename** the management virtual adapter (vNIC) for the host by running the following PowerShell command:

```powershell
$vSwitchName = "Management Team"
Rename-VMNetworkAdapter -ManagementOS -Name $vSwitchName `
    -NewName Management -CimSession $Servers
```

6. Next, you'll **create** a pair of dedicated **host storage virtual network adapters**. These will be called SMB01 and SMB02, across all the nodes in the cluster.

```powershell
$vSwitchName = "ComputeStorage Team"
foreach ($Server in $Servers) {
    # Add SMB vNICs (number depends on how many pNICs are connected to vSwitch)
    $SMBvNICsCount = (Get-VMSwitch -CimSession $Server `
            -Name $vSwitchName).NetAdapterInterfaceDescriptions.Count
    foreach ($number in (1..$SMBvNICsCount)) {
        $TwoDigitNumber = "{0:D2}" -f $Number
        Add-VMNetworkAdapter -ManagementOS -Name "SMB$TwoDigitNumber" `
            -SwitchName $vSwitchName -CimSession $Server
    }
}
```
> This command adds virtual network adapters to the host operating system, for use specifically for storage traffic. It performs this across all Azure Stack HCI nodes.

7. With that completed, you can quickly validate the host virtual network adapters that were just created:

```powershell
Get-VMNetworkAdapter -CimSession $Servers -ManagementOS
```

> This command queries the Azure Stack HCI nodes for a list of virtual network adapters that have been assigned to the host itself.

![Host virtual network adapters created](/modules/module_2/media/ps_host_vnics.png "Host virtual network adapters created")

8. Next, you'll configure the IP addresses for the storage networks to match this table. You'll create 2 separate subnets that will be applied to the dedicated host storage virtual network adapters, across all nodes.

| Node | Name | IP Address | Subnet Mask | VLAN
| :-- | :-- | :-- | :-- | :-- |
| AZSHCI1 | SMB01 | 172.16.1.1 | 24 | 1
| AZSHCI1 | SMB02 | 172.16.2.1 | 24 | 1
| AZSHCI2 | SMB01 | 172.16.1.2 | 24 | 1
| AZSHCI2 | SMB02 | 172.16.2.2 | 24 | 1
| AZSHCI3 | SMB01 | 172.16.1.3 | 24 | 1
| AZSHCI3 | SMB02 | 172.16.2.3 | 24 | 1
| AZSHCI4 | SMB01 | 172.16.1.4 | 24 | 1
| AZSHCI4 | SMB02 | 172.16.2.4 | 24 | 1

To do so, you'll run the following PowerShell code:

```powershell
$StorNet1 = "172.16.1."
$StorNet2 = "172.16.2."
$IP = 1 # Starting IP
foreach ($Server in $Servers) {
    $SMBvNICsCount = (Get-VMSwitch -CimSession $Server -Name $vSwitchName).NetAdapterInterfaceDescriptions.Count
    foreach ($number in (1..$SMBvNICsCount)) {
        $TwoDigitNumber = "{0:D2}" -f $Number
        if ($number % 2 -eq 1) {
            New-NetIPAddress -IPAddress ($StorNet1 + $IP.ToString()) `
                -InterfaceAlias "vEthernet (SMB$TwoDigitNumber)" `
                -CimSession $Server -PrefixLength 24
        }
        else {
            New-NetIPAddress -IPAddress ($StorNet2 + $IP.ToString()) `
                -InterfaceAlias "vEthernet (SMB$TwoDigitNumber)" `
                -CimSession $Server -PrefixLength 24
            $IP++
        }
    }
}
```

9. Once complete, you can validate the network configuration by running the following PowerShell:

```powershell
Get-NetIPAddress -CimSession $Servers -InterfaceAlias vEthernet* `
    -AddressFamily IPv4 | Sort-Object IPAddress |  `
    Select-Object IPAddress, InterfaceAlias, PSComputerName
```

> This command queries the Azure Stack HCI nodes for the IPv4 addresses for any network adapter with "vEthernet" in the name and returns the results.

![All IP addresses assigned to storage virtual network adapters](/modules/module_2/media/ps_ip_addresses.png "All IP addresses assigned to storage virtual network adapters")

10. Whilst not required in the nested environment, it's a best practice to configure your storage network adapters with VLANs, as QoS requires this. Here, you will assign VLANs to your host storage virtual network adapters.

```powershell
$StorVLAN1 = 1
$StorVLAN2 = 2

# Configure Odds and Evens for VLAN1 and VLAN2
foreach ($Server in $Servers) {
    $NetAdapters = Get-VMNetworkAdapter -CimSession $server -ManagementOS -Name *SMB* | Sort-Object Name
    $i = 1
    foreach ($NetAdapter in $NetAdapters) {
        if (($i % 2) -eq 1) {
            Set-VMNetworkAdapterVlan -VMNetworkAdapterName $NetAdapter.Name `
                -VlanId $StorVLAN1 -Access -ManagementOS -CimSession $Server
            $i++
        }
        else {
            Set-VMNetworkAdapterVlan -VMNetworkAdapterName $NetAdapter.Name `
                -VlanId $StorVLAN2 -Access -ManagementOS -CimSession $Server
            $i++
        }
    }
}
# Restart each host vNIC adapter so that the Vlan is active.
Get-NetAdapter -CimSession $Servers -Name "vEthernet (SMB*)" | Restart-NetAdapter
```

> The above code cycles through each of the Azure Stack HCI nodes, and assigns the individual host storage virtual network adapters to a specific VLAN ID. Half of the adapters will be assigned VLAN 1 and half to VLAN 2.

11. Once complete, you can verify the VLAN assignment by running the following PowerShell commands:

```powershell
Get-VMNetworkAdapterVlan -CimSession $Servers -ManagementOS
```

> The above code retrieves a list of VLANs associated with host virtual network adapters, across the Azure Stack HCI nodes

![All VLANs assigned to storage virtual network adapters](/modules/module_2/media/ps_vlans.png "All VLANs assigned to storage virtual network adapters")

12. In order to ensure that both SMB01 and SMB02 use **both** of the underlying pNICs in the **ComputeStorage Team**, we need to perform a mapping of the vNICs to pNICs. **Note**, this is more applicable to production environments, running with physical network adapters. However, you can perform this with the following PowerShell command in the nested environment for reference:

```powershell
Invoke-Command -ComputerName $servers -ScriptBlock {
    # Retrieve adapter names
    $physicaladapternames = (get-vmswitch $using:vSwitchName).NetAdapterInterfaceDescriptions
    # Map pNIC and vNICs
    $vmNetAdapters = Get-VMNetworkAdapter -Name "SMB*" -ManagementOS
    $i = 0
    foreach ($vmNetAdapter in $vmNetAdapters) {
        $TwoDigitNumber = "{0:D2}" -f ($i + 1)
        Set-VMNetworkAdapterTeamMapping -VMNetworkAdapterName "SMB$TwoDigitNumber" `
            -ManagementOS -PhysicalNetAdapterName (get-netadapter -InterfaceDescription $physicaladapternames[$i]).name
        $i++
    }
}

# Confirm it's completed
Get-VMNetworkAdapterTeamMapping -CimSession $servers -ManagementOS | `
    Format-Table ComputerName, NetAdapterName, ParentAdapter
```

![VM Network Adapter Team Mapping in PowerShell](/modules/module_2/media/vm_team_mapping.png "VM Network Adapter Team Mapping in PowerShell")

13. You should also check the **Jumbo Frames** setting, however for the purpose of this scenario, you'll leave it at the default setting of **disabled**, with a size of **1514**:

```powershell
Get-NetAdapterAdvancedProperty -CimSession $servers -DisplayName "Jumbo Packet"
```

![Viewing jumbo frames in PowerShell](/modules/module_2/media/vm_jumbo_frames.png "Viewing jumbo frames in PowerShell")

________________________
**NOTE** - There are a number of settings which have not been included in this section, as they do not function in a nested virtualization environment. Features such as **RDMA**, **Network QoS**, **Datacenter Bridging (DCBX)** etc. only really function correctly with the appropriate physical hardware, including compatible switches. These elements will be covered in a different module, at a later date.
________________________
