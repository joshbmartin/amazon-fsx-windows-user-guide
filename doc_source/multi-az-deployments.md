# Multi\-AZ File System Deployments<a name="multi-az-deployments"></a>

Each Amazon FSx for Windows File Server file system resides in a particular Availability Zone \(AZ\), which you specify during creation\. Amazon FSx automatically replicates file system data within the AZ, and ensures high availability within the AZ by detecting and addressing component failures\. For workloads that require multi\-AZ redundancy to tolerate temporary AZ unavailability, you can create multiple file systems in separate AZs, keep them in sync, and configure failover between them\.

Amazon FSx fully supports the use of the Microsoft Distributed File System \(DFS\) for file system deployments across multiple AZs to get Multi\-AZ availability and durability\. Using DFS Replication, you can automatically replicate data between two file systems\. Using DFS Namespaces, you can configure one file system as your primary and the other as your standby, with automatic failover to the standby in the event that the primary becomes unresponsive\. In the following topics, you can find a description of how to set up and use DFS Replication and DFS Namespaces failover across AZs with Amazon FSx\.

![\[Image NOT FOUND\]](http://docs.aws.amazon.com/fsx/latest/WindowsGuide/images/MultiAZDiagram.png)

**Prerequisites for using DFS Replication**
+ Create two Amazon FSx file systems in different AZs within an AWS Region\. For more information on creating your file systems, see [Step 3: Write Data to Your File Share](getting-started.md#getting-started-step3)\.
+ Ensure that both file systems are in the same AWS Directory Service for Microsoft Active Directory\.
+ After the file systems are created, make a note of their file system IDs\.

## Setting Up DFS Replication<a name="set-up-fsx-dfs"></a>

You can use DFS Replication to automatically replicate data between two Amazon FSx file systems\. This replication is bidirectional, meaning that you can write to either file system and the changes will be replicated to the other\.

------
#### [ Windows Server 2016 – scripted ]

**To set up DFS Replication**

1. Before you can manage DFS, you must launch the instance and connect it to the AWS Directory Service for Microsoft Active Directory to which you joined your Amazon FSx file systems\. To perform this action, choose one of the following procedures from the *AWS Directory Service Administration Guide*:
   + [Seamlessly Join a Windows EC2 Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/launching_instance.html)
   + [Manually Join a Windows Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/join_windows_instance.html)

1. Connect to your instance as a user in the **AWS Delegated FSx Administrators** group\. For more information, see [Connecting to Your Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) in the *Amazon EC2 User Guide for Windows Instances*\.

1. Download this [FSx\-DFSr\-Setup\.ps1](https://s3.amazonaws.com/solution-references/fsx/dfs/FSx-DFSr-Setup.ps1                 ) PowerShell script\.

1. Open the **Start** menu and type **PowerShell**\. From the list of matches, choose **Windows PowerShell**\.

1. Run the PowerShell script with the following specified parameters to establish DFS Replication between your two file systems:
   + The names of the DFS Replication group and folder
   + The local path to the folder you want to replicate on your file systems \(for example, `D:\share` for the default share that comes included with your Amazon FSx file system\)
   + The DNS names of the primary and standby Amazon FSx file systems you created in the prerequisite steps\.  
**Example**  

   ```
   FSx-DFSr-Setup.ps1 -group Group -folder Folder -path ContentPath -primary FSxFileSystem1-DNS-Name -standby FSxFileSystem2-DNS-Name
   ```

------
#### [ Windows Server 2016 – step\-by\-step ]

**To set up DFS Replication**

1. Before you can manage DFS, you must launch the instance and connect it to the AWS Directory Service for Microsoft Active Directory to which you've joined your Amazon FSx file systems\. To perform this action, choose one of the following procedures from the *AWS Directory Service Administration Guide*:
   + [Seamlessly Join a Windows EC2 Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/launching_instance.html)
   + [Manually Join a Windows Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/join_windows_instance.html)

1. Connect to your instance as a user in the **AWS Delegated FSx Administrators** group\. For more information, see [Connecting to Your Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) in the *Amazon EC2 User Guide for Windows Instances*\.

1. Open the **Start** menu and type **PowerShell**\. From the list of matches, choose **Windows PowerShell**\.

1. If you don't have DFS Management Tools installed on your instance already, install it with the following command:

   ```
   Install-WindowsFeature RSAT-DFS-Mgmt-Con
   ```

1. From the PowerShell prompt, create a DFS Replication group and folder with the following commands:

   ```
   $Group = Name of the DFS Replication group
   $Folder = Name of the DFS Replication folder
   
   New-DfsReplicationGroup –GroupName $Group
   Grant-DfsrDelegation –GroupName $Group –AccountName “FSxAdmins” –Force
   New-DfsReplicatedFolder –GroupName $Group –FolderName $Folder
   ```

1. Determine the Active Directory computer name associated with each file system with the following commands:

   ```
   $FileSystemId1 = File system ID of the primary FSx file system
   $FileSystemId2 = File system ID of the secondary FSx file system
                   
   $C1 = (Resolve-DnsName $FileSystemId1 –Type CNAME).NameHost
   $C2 = (Resolve-DnsName $FileSystemId2 –Type CNAME).NameHost
   ```

1. Add your file systems as members of the DFS Replication group you created with the following commands:

   ```
   Add-DfsrMember –GroupName $Group –ComputerName $C1
   Add-DfsrMember –GroupName $Group –ComputerName $C2
   ```

1. Use the following commands to add the local path \(for example, `D:\share`\) for each file system to the DFS Replication group\. In this procedure, *FileSystemID1* will serve as the primary member, meaning that its contents will initially be synced to the other file system:

   ```
   $ContentPath1 = Local path to the folder you want to replicate on file system 1
   $ContentPath2 = Local path to the folder you want to replicate on file system 2
                   
   Set-DfsrMembership –GroupName $Group –FolderName $Folder –ContentPath $ContentPath1 –ComputerName $C1 –PrimaryMember $True
   Set-DfsrMembership –GroupName $Group –FolderName $Folder –ContentPath $ContentPath2 –ComputerName $C2 –PrimaryMember $False
   ```

1. Add a connection between the file systems with the following command:

   ```
   Add-DfsrConnection –GroupName $Group –SourceComputerName $C1 –DestinationComputerName $C2
   ```

Within minutes, both file systems will begin synchronizing the contents of the `ContentPath` specified above\.

------
#### [ Earlier Versions of Windows Server ]

**To set up DFS Replication**

1. Before you can manage DFS, you must launch the instance and connect it to the AWS Directory Service for Microsoft Active Directory to which you've joined your Amazon FSx file systems\. To perform this action, choose one of the following procedures from the *AWS Directory Service Administration Guide*:
   + [Seamlessly Join a Windows EC2 Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/launching_instance.html)
   + [Manually Join a Windows Instance](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/join_windows_instance.html)

1. Connect to your instance as a user in the **AWS Delegated FSx Administrators** group\. For more information, see [Connecting to Your Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) in the *Amazon EC2 User Guide for Windows Instances*\.

1. Open the **Start** menu and type **PowerShell**\. From the list of matches, choose **Windows PowerShell**\.

1. If you don't have DFS Management Tools installed on your instance already, install it with the following command:

   ```
   Install-WindowsFeature RSAT-DFS-Mgmt-Con
   ```

1. From the PowerShell prompt, create a DFS Replication group and folder with the following command:

   ```
   $Group = Name of the DFS Replication group
                   
   New-DfsReplicationGroup –GroupName $Group
   ```

1. Delegate DFS Management Permissions for the new group you just created to the user **FSxAdmins**:

   1. Open the **Start** menu and run **dfsmgmt\.msc**\. This will open the **DFS Management** GUI tool\.

   1. Choose **Action** then **Add Replication Groups to Display**\.

   1. Choose the new group that you just created in the dialog box that opens, and choose **OK**\.

   1. Choose the new group under **Replication** in the Navigation Bar\.

   1. Choose **Action**, and then **Delegate Management Permissions**\.

   1. Type **FSxAdmins** for **Enter the object name to select**, and choose **OK**\.

   1. Navigate to the **Delegation** tab for your group, and ensure that you see the **Explicit** entry for ***your\-domain*\\FSxAdmins**\.

   1. Close the **DFS Management** GUI tool\.

1. Return to the Powershell prompt, and create a DFS Replication folder with the following command:

   ```
   $Folder = Name of the DFS Replication folder
                   
   New-DfsReplicatedFolder –GroupName $Group –FolderName $FolderNew-DfsReplicatedFolder –GroupName $Group –FolderName $Folder
   ```

1. Determine the Active Directory computer name associated with each file system with the following commands:

   ```
   $FileSystemId1 = File system ID of the primary FSx file system
   $FileSystemId2 = File system ID of the secondary FSx file system
                   
   $C1 = (Resolve-DnsName $FileSystemId1 –Type CNAME).NameHost
   $C2 = (Resolve-DnsName $FileSystemId2 –Type CNAME).NameHost
   ```

1. Add your file systems as members of the DFS Replication group you created with the following commands:

   ```
   Add-DfsrMember –GroupName $Group –ComputerName $C1
   Add-DfsrMember –GroupName $Group –ComputerName $C2
   ```

1. Use the following commands to add the local path \(for example, `D:\share`\) for each file system to the DFS Replication group\. In this procedure, *FileSystemID1* will serve as the primary member, meaning that its contents will initially be synced to the other file system:

   ```
   $ContentPath1 = Local path to the folder you want to replicate on file system 1
   $ContentPath2 = Local path to the folder you want to replicate on file system 2
                   
   Set-DfsrMembership –GroupName $Group –FolderName $Folder –ContentPath $ContentPath1 –ComputerName $C1 –PrimaryMember $True
   Set-DfsrMembership –GroupName $Group –FolderName $Folder –ContentPath $ContentPath2 –ComputerName $C2 –PrimaryMember $False
   ```

1. Add a connection between the file systems with the following command:

   ```
   Add-DfsrConnection –GroupName $Group –SourceComputerName $C1 –DestinationComputerName $C2
   ```

Within minutes, both file systems will begin synchronizing the contents of the `ContentPath` specified above\.

------

## Setting up DFS Namespaces For Failover<a name="set-up-dfs-namespace"></a>

You can use DFS Namespaces to treat one file system as your primary, and the other as your standby\. This allows you to configure automatic failover to the standby in the event that the primary becomes unresponsive\. DFS Namespaces enables you to group shared folders on different servers into a single Namespace, where a single folder path can lead to files stored on multiple servers\. DFS Namespaces are managed by DFS Namespace servers, which direct compute instances mapping a DFS Namespace folder to the appropriate file servers\.

------
#### [ GUI ]

**To set up DFS Namespaces For Failover**

1. If you don't already have DFS Namespace servers running, you can launch a pair of highly available DFS Namespace servers using the [setup\-DFSN\-servers\.template](https://s3.amazonaws.com/solution-references/fsx/dfs/setup-DFSN-servers.template) AWS CloudFormation template\. For more information on creating an AWS CloudFormation stack, see [Creating a Stack on the AWS CloudFormation Console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) in the *AWS CloudFormation User Guide*\.

1. Connect to one of the DFS Namespace servers launched in the previous step as a user in the **AWS Delegated Administrators** group\. For more information, see [Connecting to Your Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) in the *Amazon EC2 User Guide for Windows Instances*\.

1. Access the DFS management console\. Open the **Start** menu and run **dfsmgmt\.msc**\. This will open the DFS Management GUI tool\.

1. Choose **Action** then **New Namespace**, type in the computer name of the first DFS Namespace server you launched for **Server** and choose **Next**\.

1. For **Name**, type in the namespace you're creating \(for example, **corp**\)\.

1. Choose **Edit Settings** and set the appropriate permissions based on your requirements\. Choose **Next**\.

1. Leave the default **Domain\-based namespace** option selected, leave the **Enable Windows Server 2008 mode** option selected, and choose **Next**\.
**Note**  
Windows Server 2008 mode is the latest available option for Namespaces\.

1. Review the namespace settings and choose **Create**\.

1. With the newly created namespace selected under **Namespaces** in the navigation bar, choose **Action** then **Add Namespace Server**\.

1. Type in the computer name of the second DFS Namespace server you launched for **Namespace server**\.

1. Choose **Edit Settings**, set the appropriate permissions based on your requirements, and choose **OK**\.

1. Choose **Add**, type in the UNC name of the file share on the primary Amazon FSx file system \(for example \\\\*fs\-0123456789abcdef0*\.example\.com\\*share*\) for Path to folder target, and choose **OK**\.

1. Choose **Add**, type in the UNC name of the file share on the standby Amazon FSx file system \(for example, \\\\*fs\-fedbca9876543210f*\.example\.com\\*share*\) for Path to folder target, and choose **OK\.**

1. From the **New Folder** window, choose **OK**\. The new folder will be created with the two folder targets under your namespace\.

1. Repeat the last three steps for each file share you want to add to your namespace\.

------
#### [ PowerShell ]

**To set up DFS Namespaces For Failover**

1. If you don't already have DFS Namespace servers running, you can launch a pair of highly available DFS Namespace servers using the [setup\-DFSN\-servers\.template](https://s3.amazonaws.com/solution-references/fsx/dfs/setup-DFSN-servers.template) AWS CloudFormation template\. For more information on creating an AWS CloudFormation stack, see [Creating a Stack on the AWS CloudFormation Console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) in the *AWS CloudFormation User Guide*\.

1. Connect to one of the DFS Namespace servers launched in the previous step as a user in the **AWS Delegated Administrators** group\. For more information, see [Connecting to Your Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) in the *Amazon EC2 User Guide for Windows Instances*\.

1. Open the **Start** menu and type **PowerShell**\. You'll see **Windows PowerShell** from the list of matches\.

1. Context–click on **Windows PowerShell** and choose **Run as Administrator**\.

1. If you don't have DFS Management Tools installed on your instance already, install it with the following command:

   ```
   Install-WindowsFeature RSAT-DFS-Mgmt-Con
   ```

1. If you don't already have an existing DFS Namespace, you can create one using the following PowerShell commands:

   ```
   $NSS1 = computer name of the 1st DFS Namespace server
   $NSS2 = computer name of the 2nd DFS Namespace server
   
   $DNSRoot = fully qualified Active Directory domain name (e.g. mydomain.com)
   $Namespace = Namespace name you want to use
   $Folder = Folder path you want to use within the Namespace
   $FS1FolderTarget = Share path to Folder Target on File System 1
   $FS2FolderTarget = Share path to Folder Target on File System 2
   
   $NSS1,$NSS2 | ForEach-Object { Invoke-Command –ComputerName $_ –ScriptBlock { mkdir “C:\DFS\${using:Namespace}”;
   New-SmbShare –Name ${using:Namespace} –Path “C:\DFS\${using:Namespace}” } }
   
   New-DfsnRoot -Path "\\${DNSRoot}\${Namespace}" -TargetPath "\\${NSS1}.${DNSRoot}\${Namespace}" -Type DomainV2
   New-DfsnRootTarget -Path "\\${DNSRoot}\${Namespace}" -TargetPath "\\${NSS2}.${DNSRoot}\${Namespace}"
   ```

1. To create a folder within your DFS Namespace, you can use the following PowerShell command\. This will create a folder that directs compute instances accessing the folder to your primary Amazon FSx file system by default:

   ```
   $FS1 = DNS name of primary FSx file system
   New-DfsnFolder –Path “\\${DNSRoot}\${Namespace}\${Folder}" -TargetPath “\\${FS1}\${FS1FolderTarget}” –EnableTargetFailback $True –ReferralPriorityClass GlobalHigh
   ```

1. You can now add your standby Amazon FSx file system to the same DFS Namespace folder\. Compute instances accessing the folder will fall back to this file system in the event that they can't connect to the primary Amazon FSx file system:

   ```
   $FS2 = DNS name of secondary FSx file system
   New-DfsnFolderTarget –Path “\\${DNSRoot}\${Namespace}\${Folder}" -TargetPath “\\${FS2}\${FS2FolderTarget}”
   ```

You can now access your data from compute instances using the DFS Namespace folder’s remote path specified preceding\. This then directs the compute instances to the primary Amazon FSx file system \(and to the standby file system, if the primary is unresponsive\)\.

For example, open the **Start** menu and type `PowerShell`\. From the list of matches, choose `Windows PowerShell` and run the following command\.

```
net use Z: \\${DNSRoot}\${Namespace}\${Folder} /persistent:yes
```

------