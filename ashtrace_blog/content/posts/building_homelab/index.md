+++
title = 'AD Homelab 101: Building an Active Directory + XDR homelab in 101 steps'
date = 2025-05-23T13:40:31+05:30
tags = ["active directory", "homelab"]
+++

Fresh off my CRTO exam I itched for a local AD lab to practice red-team stuff. GOAD with the ELK/Wazuh extension is (at the time of writing) the best choice (author’s personal views) for this but I seriously lacked a gigaton of RAM required for 5 (lab) + 1 (extension) VMs, so I went Thanos mode and declared - Fine! I’ll do it myself.

> PS: I’ve tried GOAD (v2) without the ELK/Wazuh stack. It’s a stellar lab that hits a lot of awesome topics. If you haven’t touched it yet, crawl out from under that rock and go check it out.

## My host device specifications

The machine I used to built this lab has

- An AMD Ryzen 5500U processor - 6 Physical (12 Logical) cores
- 32 gigs of RAM (which might’ve been huge for ancient times, but since the dawn of LLMs I feel smol and scared.)


## Domain Controller

### Fetch Windows Server Image

Download the Windows Server 2022 VHD file from [here](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022). Create a copy of the downloaded file.

### Setup a VM

I used VirtualBox to create a new VM named LAB-DC and imported the downloaded VHD file. I provided the lab with Virtualbox Host-Only network connection, which can be created leveraging the `Tools > Network` window.

![virtualbox-network](./virtualbox-network.png)

Boot up the VM and follow the installation steps:

1. Select the Locale settings.

![locale](./locale.png)

2. Accept the License agreement.

3. Create a password for the administrator account. I used `lab@1234$`.

![administrator-password](./administrator-password.png)

4. Click on Finish.

### Configure VM addons

5. Log-in as administrator.

6. From the toolbar, select add 'Virtualbox Guest-Addons'. Install it to improve the VMs execution.

![vbox-guest-addons-1](./vbox-guest-addons-1.png)

![vbox-guest-addons-2](./vbox-guest-addons-2.png)

Follow along after the server VM restarts.

### Set the server hostname

7. In the server manager application window, navigate to `Local Server` from the left pane. Click on `Computer Name` under `Properties`. Further, click on the `Change` button to change server name.

![server-name-1](./server-name-1.png)

![server-name-2](./server-name-2.png)

Click on `OK` and restart the VM.

### Configure the static IP address

8. Hit `Win+R` and run `ncpa.cpl` to open Network Connections under Control Panel. View the property of the ethernet interface.

![static-ip-ethernet-properties](./static-ip-ethernet-properties.png)

9. Click on `Internet Protocol Version 4 (TCP/IPv4)` and select its properties. Set the preferred IP Address and set the gateway to Host's IP Address. Set the preferred DNS same as the IP Address of the server. Finally, click on `OK`.

![static-ip-ipv4-properties](./static-ip-ipv4-properties.png)

## Install Active Directory Domain Services

10. Launch `Server Manager`. Navigate to `Manage > Add Roles and Features`. Continue with default installation.

![server-manager-manage-add-roles-and-features](./server-manager-manage-add-roles-and-features.png)

![add-roles-and-featuers-begin](./add-roles-and-featuers-begin.png)

11. Continue Clicking on next until `Select Server roles`. Select `Active Directory Domain Services` and click on `Add Features` in the window that pops-up.

![add-roles-and-featuers-select-server-roles](./add-roles-and-featuers-select-server-roles.png)

Ensuring that `Active Directory Doman Services` is selected, click on `Next`.

12. Go forward with default selection in `Features` and `AD DS` tab by clicking on `Next`.

13. On the `Confirmation` tab, select the `Restart the destination server automatically if required` if required and click on `Install`.

![add-roles-and-features-confirmation](./add-roles-and-features-confirmation.png)

14. Once greeted with following screen, click on the flag icon.

![add-roles-and-features-install-completed](./add-roles-and-features-install-completed.png)

Under Post-deployment configuration section, click on `Promote this server to a domain controller`.\

![post-deployment-configuration-section](./post-deployment-configuration-section.png)

15. Create a new forest and set the appropriate name.

![post-deployment-create-forest](./post-deployment-create-forest.png)

Click on next.

16. Let the forest functional level and domain functional level be at default of `Windows Server 2016`.

![post-deployment-functional-level](./post-deployment-functional-level.png)

17. Keep the default roles for the DC. Set the DSRM password. I used `labdsrm@1234$`

![post-deployment-dsrm](./post-deployment-dsrm.png)

18. In the `DNS Options` menu, just click `Next`.

![post-deployment-dns](./post-deployment-dns.png)

19. Follow through and click `Next` on verify the NetBIOS domain name.

20. Under the `Paths` configuration window go with default settings if no changes are needed.

![post-deployment-paths](./post-deployment-paths.png)

21. Click on `Next` under `Review Options` window.

22. After `Prerequisite Checks` pass, click on `Install` to continue.

![post-deployment-prerequisite-checks](./post-deployment-prerequisite-checks.png)

23. Let the server restart.

![post-deployment-installation-complete](./post-deployment-installation-complete.png)

24. Once the server restarts, log in as `<DOMAIN>\Administrator` using the password of administrator we setup above while installing the VM (here: `lab@1234$`).

## Creating Domain Objects

### Creating Organizational Units (OUs)

> **NOTE:** One may skip this and directly add users.

25. Launch Server Manager and navigate to `Tools > Active Directory Users and Computers`

![ad-users-and-computers](./ad-users-and-computers.png)

26. Right-click on the `<DOMAIN NAME>` (here and hereafter `LAB.LOCAL` for us), select `New` and click on `Organizational Unit`.

![ad-new-ou](./ad-new-ou.png)

In the dialog bog, provide with an OU name of your choice and click on `OK`.

![ad-new-ou-lab-ou](./ad-new-ou-lab-ou.png)

We can create nested OUs by:
- Right-click on the OU of choice, navigate to `New > Organizational Unit` and follow the process as above.

I created two more OUs under our `LAB-OU` namely, `Users` and `Computers`, within `Users` I further created two OUs - `Administrators` and `Researchers`. (It made for a good practice)

![nested-ous](./nested-ous.png)

### Creating a User

27. Right-click the `Administrators` OU under `LAB-OU > Users`, navigate to `New > User`.

28. Fill in the name details. Click on `Next`.

![add-user-name](./add-user-name.png)

29. Set the password details and select any other configuration required (here: `admin@1234`). Click on `Next`.

![add-user-passwd](./add-user-passwd.png)

30. Click on `Finish`.

> Practise by creating multiple users.

### Promoting a user to Domain Administrator

31. Right-click on the newly created user and navigate to `Properties`.

![user-properties](./user-properties.png)

32. Navigate to `Member Of` Tab. The screen should display the groups this particular user is part of.
Click on `Add...`.

![user-member-of](./user-member-of.png)

33. In the `Select Groups` window, within the form-field labeled **Enter the object names to select (examples)**, enter `domain` and Click on `OK`.

![select-da-group-1](./select-da-group-1.png)

A dialog box with all group names starting with `Domain` should appear, select the `Domain Admins` group and click on `OK`.

![select-da-group-2](./select-da-group-2.png)

Click on `Apply`, then click on `OK`.

### Creating a Group

34. Navigate to the OU of your choice, right-click and select `New > Group`. Add details and click on `OK`.

![add-group](./add-group.png)

### Add Group Members

35. Right-click on the newly created group and select `Properties`. Open the `Members` Tab, click on `Add...`.

![group-members-1](./group-members-1.png)

36. A `Select Users, Contacts, Computers, Service Accounts, or Groups` window opens up. Within the form field under **Enter the objct names to select (examples)** enter the name of target user (click on `Check Names` to correct the format). Finally, click on `OK`.

![group-members-add-user](./group-members-add-user.png)

Click on `Apply` and then click on `OK`.

> Group membership of user can be verified by navigating to user's `Properties > Member Of`.

## Creating a File Share

37. Create a new folder (here `test-share`).

![new-folder-test-share](./new-folder-test-share.png)

38. Navigate to `Properties > Sharing` from the context-menu of the folder. Click on `Advanced Sharing`.

![test-share-sharing-tab](./test-share-sharing-tab.png)

39. Enable `Share this folder`. Configure `Share name` if needed, click on `Apply` and click on `OK`.

![test-share-advanced-sharing](./test-share-advanced-sharing.png)

40. Visit the DC and the file-share would be visible.

![run-lab-dc](./run-lab-dc.png)

![lab-dc-files-shares](./lab-dc-files-shares.png)

### Configure file-share permissions

41. Navigate to `Properties > Security` Tab from context-menu of the folder/file share.

![test-share-security-tab](./test-share-security-tab.png)

42. Click on `Edit`. Next, click on `Add...`

![permissions-for-test-share](./permissions-for-test-share.png)

43. In the `Select Users, Computers, Service Accounts, Grups` window, search and select the entities. Click on `OK`.

![test-share-select-user](./test-share-select-user.png)

44. Change the Permissions for the entity from the `Allow`/`Deny` list. Finally, Click on `Apply` and `OK` respectively.

![test-share-user-permissions](./test-share-user-permissions.png)

## Adding a computer to the AD Domain

A long time ago in a galaxy far, far away Microsoft offered Windows VM images to test Internet-Explorer. The files have since been archived across internet. One may grab the version that suits them [here](https://archive.org/download/modern.ie-vm) all other places of their choice. I am using virtualbox, so it would be a virtualbox image in my case.

45. Download the archive, extract it and import the (here `.ova`) file in virtualbox (`Ctrl+I` for importing an image).

![ova-file](./ova-file.png)

![ova-import](./ova-import.png)

> Configure the network adapter of new machine to connect to Virtualbox host-only adapter (same as the DC).

46. Log into the VM (`IEUser:Passw0rd!`). Run (`Win+R`) `ncpa.cpl` to enter `Network Connections` window in `Control Panel > Network and Interent`. Configure the DNS server to DC IP for the Ethernet interface by navigating through `Properties` (as above).

Although the Virtualbox host-only adapter would provide this VM with a range in same subnet as DC through DHCP, I will configure a static IP for this machine (for identification purposes in my later projects).

![machine-network](./machine-network.png)

47. Launch `Settings` and go to `Accounts > Access work or school`. Click on `Connect`.

![settings-add-account](./settings-add-account.png)

48. Click on `Join this device to a local Active Directory domain`. 

![join-to-local-ad](./join-to-local-ad.png)

49. Enter the domain name in the `Join a domain` window. Click on `Next`.

![join-a-domain](./join-a-domain.png)

50. Enter the username and password of a Domain Administrator account to authenticate.

![login-in-ad](./login-in-ad.png)

51. Click on `Skip`.

![skip-rest](./skip-rest.png)

Restart the VM.

52. Login using one of the user accounts created earlier.

![user-login-ad](./user-login-ad.png)

53. Go to `Control Panel > Network and Interent > Network and Sharing Center` to validate you are connected to the domain.

![network-and-sharing-center](./network-and-sharing-center.png)

The reachability can be established from the command-prompt as follows:

![cmd-domain-reachability](./cmd-domain-reachability.png)

> The computer would be visible in the `Computers` Section under the `Active Directory Users and Computers`. It can be *dragged-and-dropped* to any OU we created.

![domain-joined-computer](./domain-joined-computer.png)

## Create a Group Policy

54. Back on the DC machine, launch the `Server Manager` and go to `Tools > Group Policy Management`.

![group-policy-management](./group-policy-management.png)

One can navigate through different Organizational Units (OUs), and select the one required.

![gpm-ous](./gpm-ous.png)

### Group policy to create a local administrator account

55. Right click on the OU with computers, select `Create a GPO in this domain, and Link it here...`

![create-a-gpo-and-link-it-here](./create-a-gpo-and-link-it-here.png)

56. Provide a name for the new Group Policy Object (GPO) and click on `OK`.

![new-gpo-name](./new-gpo-name.png)

57. Right-click on the newly created GPO and select `Edit`.

![edit-gpo](./edit-gpo.png)

58. A `Group Policy Management Editor` window pops-up. Go to `Preferences > Control Panel Settings` and select `Local Users and Groups`. Right-click on the table (empty here), and select `New > Local Group`.

![local-users-and-groups](./local-users-and-groups.png)

59. Within the `New Local Group Properties`, set the `Action` to be `Update`. Select the `Group name` to be `Administrators (built-in)` from the drop-down menu.

![groups-drop-down](./groups-drop-down.png)

60. Click on `Add` under `Members` table.

![add-groupmember](./add-groupmember.png)

61. Click on the `...` button beside name to spawn the `Select User, Computer, or Group` window.

![local-group-member-add-3-button](./local-group-member-add-3-button.png)

62. Enter the object name (here `ashtrace`, a domain-joned user I created earlier). Click on `Check Names` to retrieve the particular user name in correct format. Click on `OK`.

![select-user-ashtrace](./select-user-ashtrace.png)

Click on `OK` again (Notice that the user name is prefixed by the domain name.)

![select-user-ashtrace-2](./select-user-ashtrace-2.png)

Finally, Click on `Apply` and `OK` respectively.

63. Validate the GPO by going back to `Group Policy Management` Window, navigate to the OU and select the GPO created. Visit the `Settings` tab.

![gpo-settings-page](./gpo-settings-page.png)

Titles can be expanded through clicks over them, and it can be observed that the GPO updates the membership of the built-in administrators group.

![gpo-settings-page-2](./gpo-settings-page-2.png)

> A window-pop with alert from internet-explorer/edge might spawn complaining website trust issues as the setting page renders an HTML file. Go forth and trust the source, by selecting `Add` in the window itself, to render the contents.

### Syncing group policy updates

64. Go to the machine added earlier.

65. Either reboot it, or spawn a command prompt and type `gpupdate /force`.

65. Once update, open up file explorer. Right-click on `This PC` and select `Manage`.

![this-pc-manage](./this-pc-manage.png)

Within the `Computer Management` window, go to `Local Users and Groups`. Double-click on `Groups`, followed by a `double-click` on `Administrators`.

![administrators-group-members](./administrators-group-members.png)

It is evident that `LAB\ashtrace` is a member of the builtin-administratosr group now, and their credential can be used to exeute task with administrative privileges.

## Add a Server to AD Lab

66. Use the copy of the Windows server VHD image we created earlier, to spawn a new VM machine connected to the Virtualbox Host-Only adapter.

> **NOTE**: If you get UUID conflict run `C:\Users\ashtrace\VMs>"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" internalcommands sethduuid <path-to-vhd-file>`

67. Configure the locale settings and setup the administrator password (reference the steps executed when preparing VM for the DC i.e. steps 1-2-3-4).

68. Once the VM reboots, through the server manager configure a hostname for the VM (reference step 7). I named the server `LAB-SRV`.

69. After the VM restarts again, configure a static IP address and enter the DC's IP for DNS (reference step 46, use credential setup for lab server to login).

![lab-srv-ip](./lab-srv-ip.png)

70. Open file explorer. Right-click on `This PC`, select `Properties`. This opens up the `About` page in `Settings`. Scroll-down and click on `Advanced system settings`. In the pop-window go to `Computer Name` tab and click on `Change`.

![lab-srv-connect-to-domain](./lab-srv-connect-to-domain.png)

71. Switch to `Domain` from `Workgroup` and enter the domain name (here: `LAB.LOCAL`), enter the domain administrator credentials. After successful authentication a dialog box with message `Welcome to domain <domain name>` should appear. Restart the VM when asked.

![lab-srv-connect-to-domain-auth](./lab-srv-connect-to-domain-auth.png)

72. After reboot, use domain credentials to log onto the server. I used the `ashtrace` account credentials as it would be part of the built-in administrator group owing the GPO created earlier (Ensure that the LAB-SRV machine is part of the OU to which the GPO has been mapped, if not on the DC, open up `Server Manager` > go to `Active Directory Users and Computers` from `Computers` drag the `LAB-SRV` machine to the particular OU (`LAB-OU > Computers` in this case), back on the `LAB-SRV` machine run `gpupdate /force` and reboot)

### Configure IIS service on the newly added server

> We briefly enable a second network adapter for the VM and allow access to the Wi-fi network.

![bridged-network-lab-srv](./bridged-network-lab-srv.png)

73. In the `Server Manager` Application, select to `Manage > Add Roles and Features`. Select `Role-based or feature-based installation` mode and click on `Next`. Ensure that the server is selected in `Server Selection` window.

74. Select `Web Server IIS` in the `Server Roles` window and click on `Add features` in the pop-window that appears. Click on `Next`.

![web-server-iis-role](./web-server-iis-role.png)

75. Click on `Next` in `Features` window followed by another `Next`. In the `Web Server Role (IIS)` window's `Role services` list select following under `Application Development` (if you want ASP.NET support) along with the default features selected. Click on `Next`.

![web-server-iis-roles-services](./web-server-iis-roles-services.png)

76. Under `Confirmation` allow the wizard to restart the VM if needed and click on `Install`.

77. Once the installation succeeds, visit the server from the workstation added earlier to establish if the IIS service is up and running.

![iis-default-page-on-lab-srv](./iis-default-page-on-lab-srv.png)

## Adding XDR

Wazuh is an open-source XDR. It ships [OVA](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html) image among other installation methods.

78. Download the OVA file and import it in Virtualbox.

79. Change the network adapter to host-only, change the graphics controller to `VMSVGA` (under `Settings > Display > Graphics Controller`) and enable the `Enable Hardware Clock in UTC Time` feature under `Settings > System > Extended Features`.

80. Power-up the VM, login using credentials `wazuh-user:wazuh`.

### Setup static IP address

81. Find the name of the ethernet interface using `ip a`.

```
[wazuh-user@wazuh-server ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:bf:d2:7c brd ff:ff:ff:ff:ff:ff
    altname enp0s17
    inet 192.168.56.107/24 brd 192.168.56.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:febf:d27c/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```

82. Check the status of the ethernet interface to identify the file being used to manage it from `systemd-networkd` by running `networkctl status <interface-name>` (here and hereafter the interface-name is `eth0`)

```
[wazuh-user@wazuh-server ~]$ networkctl status eth0
● 2: eth0
                     Link File: /usr/lib/systemd/network/99-default.link
                  Network File: /etc/systemd/network/10-cloud-init-eth0.network
                         State: routable (configured)
                  Online state: online
                          Type: ether
                          Path: pci-0000:00:11.0
                        Driver: e1000
                        Vendor: Intel Corporation
                         Model: 82545EM Gigabit Ethernet Controller (Copper) (PRO/1000 MT Single Port Adapter)
             Alternative Names: enp0s17
              Hardware Address: 08:00:27:bf:d2:7c (PCS Systemtechnik GmbH)
                           MTU: 1500 (min: 46, max: 16110)
                         QDisc: fq_codel
  IPv6 Address Generation Mode: eui64
      Number of Queues (Tx/Rx): 1/1
              Auto negotiation: yes
                         Speed: 1Gbps
                        Duplex: full
                          Port: tp
                       Address: 192.168.56.107 (DHCP4 via 192.168.56.100)
                                fe80::a00:27ff:febf:d27c
             Activation Policy: up
           Required For Online: yes
               DHCP4 Client ID: IAID:0x62b7eef0/DUID
             DHCP6 Client DUID: DUID-EN/Vendor:0000ab116c1ecb03a0526b40
```

83. Check the value of `Network File` attribute (here: `/etc/systemd/network/10-cloud-init-eth0.network`). Create a file with name `<words-between-number-and-interface>.disabled` (eg: `cloud-init.disabled`)

```
[wazuh-user@wazuh-server ~]$ sudo touch /etc/cloud/cloud-init.disabled
```

84. Remove the file discovered as value of `Network File` attribute.

```
[wazuh-user@wazuh-server ~]$ sudo rm /etc/systemd/network/10-cloud-init-eth0.network
```

85. Create a new file `/etc/systemd/network/10-static.network`

```
[Match]
Name=eth0

[Network]
Address=192.168.56.150/24
Gateway=192.168.56.1
DNS=8.8.8.8
DNS=1.1.1.1
```

86. Restart the `systemd-networkd` service.

```
sudo systemctl restart systemd-networkd
```

87. Validate the changes through `networkctl status eth0`.

### Access the Wazuh dashboard

88. Visit the URL through IP address configured earlier (here: `https://192.168.56.107`) and credential `admin:admin`.

![wazuh-dashboard](./wazuh-dashboard.png)

### Deploy Wazuh agents

89. Click on `Agent Management > Summary` from the hamburger menu icon on the top left of the dashboard. For your first agent `Deploy new agent` page should appear, further agents can be added via clicking on `Deploy new agent` button.

![deploy-new-agent-first-wazuh-agent](./deploy-new-agent-first-wazuh-agent.png)

90. Under `Select the package to download and install on your system:` select `Windows: MSI 32/64 bits`. Under `Server address`, add the add IP address configured for this wazuh-server VM.

![deploy-new-agent-windows-installer-1](./deploy-new-agent-windows-installer-1.png)

91. Skip optional settings, the command under `Run the following commands to download and install the agent:` fetches the installer from `packages.wazuh.com` thus for the sake of installation enable a second network adapter on the VMs and provide access to internet through `Bridged Adapter`.

![bridged-adapter](./bridged-adapter.png)

92. Log onto the target VM with administrative account (Domain administrator on `LAB-DC` and local administrator, for e.g. the one created earlier through GPO, on the workstation and server `LAB-SRV`). Open a PowerShell session with administrative privileges. Copy the command from `Run the following commands to download and install the agent:` section of the `Deploy new agent` page and run it in the PowerShell session.

![wazuh-agent-installation-command-powershell](./wazuh-agent-installation-command-powershell.png)

93. After the command is executed run `NET START WazuhSvc` from the same powershell session. Through `Task Manager > Services` it is evident a certian `WazuhSvc` service was created and is running.

![wazuh-svc](./wazuh-svc.png)

After a while the agent would be visible on the `Agent Management > Summary` dashboard (first as `Never connected` then as `Active`.)

![agent-management-summary-dashboard-agent-active](./agent-management-summary-dashboard-agent-active.png)

94. Repeat the steps to install the agent on other VM machines in the `LAB.LOCAL` domain. Finally, the `Agent Management > Summary` should look like this.

![agent-management-summary-dashboard-all-agents](./agent-management-summary-dashboard-all-agents.png)

## Finally

95. Agument this lab setup by adding on other servers and configurations like delegation, PKI etc. and do share the guides and resources with me.

96. Eat

97. Sleep

98. Hack

99. Repeat

100. Stay Hydrated

101. Touch Grass

## Quick Troubleshoot

- The log of wazuh agent can be viewed from the file `C:\Program Files (x86)\ossec-agent\ossec.log`
