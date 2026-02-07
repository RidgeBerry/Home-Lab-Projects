# Windows Server Home Lab & Active Directory Deployment

---

<aside>
Table of Contents

</aside>

# **Introduction**

This home lab project served as an introduction to Windows Server Administration. For this project I followed a YouTube course by East Charmer where I was walked through how to install Active Directory, how to use Active Directory, the basics of Group Policy Management, and the basics of secure file sharing.

Here is the link to the playlist: [https://www.youtube.com/playlist?list=PLAdEnQWAAbfXMY2D4HVZOe-ChfTKmaJfQ](https://www.youtube.com/playlist?list=PLAdEnQWAAbfXMY2D4HVZOe-ChfTKmaJfQ)

There is also a pdf of the steps :[https://www.eastcharmer.com/resources/0111e60a-de65-4bfd-9914-1be1f83b5093](https://www.eastcharmer.com/resources/0111e60a-de65-4bfd-9914-1be1f83b5093)

## Objective

To build a functional Windows Server 2022 environment to manage users, centralize authentication, and implement secure file sharing

## Project Overview & Architecture

This home lab was done on my Proxmox VE Host. The lab consisted of two VMs, one with Windows Server 2022 OS, and one client with Windows 11 Pro

- Hypervisor: Proxmox VE
- Virtual Machines:
    - Domain Controller(DC): Windows Server 2022
    - Client Machine: Windows 11 Pro
- Network: The Domain Controller was configured with a manual static IP address, and the client machine was given an IP through DHCP

## Implementation Steps

- Active Directory Domain Services
    - Role Installation: Used the Server Manager Dashboard to add the “Active Directory Domain Services” role.
    - Domain Promotion:  Promoted the server to a Domain Controller for a new forest (WindowsServer.local)
    - Server Naming: Renamed the server from the random default to something standard DC01 to make management easier
- Organizational Structure & Users
    - OU Design: Created Organization Units (OUs) for USA, Europe, and Asia. Each OU has sub OUs of Users, Computers, and Servers.
    - Sub OU Design: Created OUs for each department I wanted to simulate, IT, HR, Sales, Marketing, etc.
    - Security Groups: Created “Global Security Groups” for each department, (IT_dept, HR_dept, etc)
    - User Creation: Made three user accounts for each department and made them members of the correct security groups. Named after Fallout Characters.
- Group Policy Object
    - Security & Account Hardening
        - Password Policy: Configured at the Domain Level to enforce a minimum length of 12 characters and the basic complexity requirements
        - Account Lockout: Configured at the Domain Level, set a threshold of 3 failed attempts to protect against brute force attacks. If a user exceeds this, the account is automatically locked for 30 mins
    - User Environment
        - Automated Drive Mapping: Instead of manual setup, I used GPO preferences to automatically map the S: drive to the SHARED folder on DC01 for all authenticated users
    - Desktop Restrictions:
        - Control Panel Access: Disabled Access to the Control Panel and PC settings for standard users.  Do not add this at the domain level because it will block admins out of the control panel too.
- Client Integration:
    - Server’s IP: Manually set the Windows Server IP to a static IP address.
    - DNS Alignment: Manually set the Client PC’s  DNS to the Server’s IP.
    - The Join: Made the Client join the domain using the domain name WindowsServer.local and used Admin credentials to Join the Domain.
    - Verification: Logged into the Client PC using the calvinberry domain account that I created.
- Secure File Sharing:
    - Folder Creation: Created a “SHARED” Folder on DC01
    - Shared Permissions: Set to “Everyone: Full Control” at the share level
    - NTFS Permissions: I restricted access so that only admins would have full control while normal users could only read from the folder
    - Mapping: Created a GPO that would map the SHARED folder to the S: drive automatically when a user logged in
- Blocking Ringo from Accessing a Folder
    - Folders Created: Created a “Project Folder” on DC01 which contained a confidential sub folder and a materials sub folder
    - Shared Permissions: Set to “Everyone: Full Control” at the share level
    - NTFS Permissions: Set to “Everyone: Full Control” on the main project folder, but on the “Confidential” sub folder I turned off inheritance and added the user Ringo to the users list and denied read to the “Confidential” folder
    - Verification: Logged into Ringo and tried to open the folder and could not
- Other File Sharing Tests:
    - I did a few more of these kind of folder sharing examples where I would only allow a certain security group to access certain folders.
- Access Based Enumeration:
    - Folders Created: Created a main DeptShares folder with IT,HR sub folders
    - Folder Permissions: Allowed both departments to access the DeptShares folder but restricted access to the sub folders so that only users from the IT dept could access the IT folder and HR users could only access the HR folder
    - Enabling ABE:
        - Went to File and Storage Services in the Sever Manager
        - Went to Shares
        - Right Clicked the DeptShares Folder and went to properties
        - Went to Settings and checked the box next to enable access-based enumeration
    - Verification: Logged into an IT user and went to the folder and I could not see the HR dept folder, then I logged into an HR user and went to the folder but this time not seeing the IT folder

## Lessons Learned/Trouble Shooting

- When setting up the GPOs I first had them all at the domain level. At first this wasnt a problem because the first few GPOs were harmless like basic drive mapping and password policies. That changed when I added the Restrict Control Panel Access policy this policy restricts users from accessing the control panel as well as pc settings. This caused a huge problem when I was trying to set up a static IP address for DC01 and it took me a little bit to figure out why. So note to self be careful of where you place GPOs because they could lock admins out of things that only standard users should be restricted from.

---
