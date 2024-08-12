---
layout: single
title: "Red Team Scenario Lab - 01"
excerpt: "Setting up your own Lab to test (and patch) Red Team Scenarios"
date: 2024-08-12
header:
  thumb: /assets/images/lab01/in_tha_lab.jpg
  teaser: /assets/images/lab01/in_tha_lab.jpg
  teaser_home_page: true
  classes: wide
categories:
  - Lab Setup
tags:
  - Red Team
  - MSSQL
---

## Red Team Scenario Lab

Recently, I was tasked with setting up an internal environment to test and emulate some red team scenarios and TTPs. My instructions were to create something reproducible, stable, and impactful—without causing any "oops, I bricked it" moments or needing constant admin elevation. Basically, we want a lab that people can play in but doesn’t require anyone to call IT (me) for a rescue mission.

![Lab Setup](/assets/images/lab01/me_myteam.png)

This has a secondary effect of encapsulating high impact scenarios enabled by humble access levels - I think this is COOL 😎 and I will endeavour to keep this a running theme throughout this series. 

To keep this fun and engaging (both for the team but also myself) I have opted to use the fictional [Umbrella Corporation](https://residentevil.fandom.com/wiki/Umbrella_Corporation) as our persistent target, and created an internal Intranet and Dev Intranet sites containing vulnerable configurations, scripts to setup characters and groups in our Active Directory inline with the lore, aswell as scripts to create and populate databases with dummydata to reflect the Resident EVil universe - these will all be provided during the steps to create this lab (yes I am a nerd with no life GG)

As this is the first iteration, this lab has been set up to be extremely basic and we are essentially boiling the potatoes by spinning up the infra - the meat will come later. 

<img src="https://github.com/kymb0/kymb0.github.io/blob/master/assets/images/lab01/quagmire.jpg" alt="Quagmire" width="300"/>


The groundwork we put in here however will be the foundation for future scenarios and we will be modifying it accordingly, becoming a bit more secure and complex as time marches on. 
Right now however, we will not concern ourselves with implementing Endpoint Detection and Response (EDR) or OS hardening; these will be slowly introduced as these labs evolve each quarter, blending evasion, C2 usage, and scenarios together.

## WHY THO

Well, the idea is to be able to spin up and present bite-sized real world scenarios for the team (and yourself) to "drill", much like a SWAT team drills kicking down doors to taze your grandma after she stole a can of beetroot from woolies (AGAIN) - we will be drilling opening command prompt and running a `.bat` that displays matrix rain 🕶️


## LAB01 Environment Description 

Participants will interact with a simulated corporate intranet that includes both a production and a less-secure development version. This setup mirrors common real-world scenarios where development environments are often less protected or contain outdated configurations that reveal critical vulnerabilities. 

- **Single-Host Focus**: All actions and exploits are designed to be carried out from a single host, emphasizing the potential for extensive damage even from a limited foothold.
- **Non-Destructive Tactics**: The lab emphasizes using non-destructive methods to achieve impactful results, highlighting the importance of stealth and precision in professional penetration testing.
- **Real-World Relevance**: The scenarios are crafted to reflect common security challenges in real corporate environments, focusing on the exploitation of existing privileges and access rather than seeking new vulnerabilities.

## Demonstrated Skills

- **Error Message Analysis**: Analyze and leverage information from error messages to guide further exploration and exploitation.
- **Security Configuration Flaws**: Identify and exploit common security flaws found in development environments that can lead to broader network compromise.
- **Weaponise Sensitive Data Handling**: Capitalise on the risks associated with the exposure of sensitive configuration data.
- **Effective Use of Legitimate Access**: Understand how to use valid credentials to explore and extract data from networked systems, demonstrating the potential for significant impacts even without privilege escalation.


---

## Instruction Index

1. [Deploying Virtual Machines](#1-deploying-virtual-machines)
2. [Setting Up the Domain Controller (DC01)](#2-setting-up-the-domain-controller-dc01)
3. [Setting Up Client and Additional Servers](#3-setting-up-client-and-additional-servers)
4. [Setting Up Group Managed Service Accounts (gMSAs)](#4-setting-up-group-managed-service-accounts-gmsas)
5. [Create the gMSAs for DB01 and DB02](#5-create-the-gmsas-for-db01-and-db02)
6. [Installing SQL Server](#6-installing-sql-server)
7. [Creating and Configuring Database Permissions and Accounts](#7-creating-and-configuring-database-permissions-and-accounts)
8. [Create Web App](#8-create-web-app)


### 1. Deploying Virtual Machines

1. **Deploy VMs**: 
   - Create VMs for `DC01`, `Client01`, `DB01`, `DB02`, and `WEB01` 
   - Install Windows Server 2019 on `DC01`, `DB01`, `DB02`, and `WEB01`.
   - Install Windows 10 on `Client01`.

### 2. Setting Up the Domain Controller (DC01)

1. **Rename and Configure DC01**:
   - Assign the IP address `192.168.66.1` to DC01
   - Rename the server to `DC01`.

2. **Install and Promote Active Directory Domain Services**:
   - Install the AD, ADFS, and DNS role.
   - Promote the server to a domain controller and create a new forest named `umbrellacorp.local`.

### 3. Setting Up Client and Additional Servers

1. **Rename and Configure Network Settings**:
   - Assign the IP addresses statically as follows, make sure the DC ip is set as the preffered DNS:
     - `Client01`: `192.168.66.20`
     - `DB01`: `192.168.66.10`
     - `DB02`: `192.168.66.11`
     - `WEB01`: `192.168.66.12`

   -**Create DNS entries for Intranet Pages**
      - On the domain controller, create an A record for `intranet.umbrellacorp.local` and `dev.env.intranet.umbrellacorp.local` and point them to `WEB01`

   -**Populate Acitive Directory with lore-friendly nerds**
      - On the domain controller, run [this](https://github.com/kymb0/bucket/blob/main/create_groups_users.ps1) script that will create security groups and suers which the script will then add the users into

3. **Join the Domain**:
   - On each server (`Client01`, `DB01`, `DB02`, `WEB01`), go to **System Properties** and change the settings to join the `umbrellacorp.local` domain.
   - Restart each VM after joining the domain.

### 4. Setting Up Group Managed Service Accounts (gMSAs)

1. **Prepare the Domain for gMSA**:
   - On DC01, open an elevated PowerShell prompt and create the KDS root key:
   ```powershell
   Add-KdsRootKey -EffectiveTime (Get-Date).AddHours(-10)
   ```

### 5. Create the gMSAs for DB01 and DB02

- **Prepare the Key Distribution Services Root Key**:
  - Run on DC01 (Domain Controller):
    ```powershell
    Add-KdsRootKey -EffectiveTime (Get-Date).AddHours(-10)
    ```
- **Create Group Managed Service Accounts**:
  - Run on DC01 (Domain Controller):
    ```powershell
    New-ADServiceAccount -Name "gmsa_db01" -DNSHostName "DB01.umbrellacorp.local" -PrincipalsAllowedToRetrieveManagedPassword "DB01$"
    New-ADServiceAccount -Name "gmsa_db02" -DNSHostName "DB02.umbrellacorp.local" -PrincipalsAllowedToRetrieveManagedPassword "DB02$"
    ```

- **Install gMSAs on Servers**:
  - Run on DB01:
    ```powershell
    Install-WindowsFeature RSAT-AD-PowerShell
    Import-Module ActiveDirectory
    Install-ADServiceAccount gmsa_db01
    ```
  - Run on DB02
   ```powershell
    Install-WindowsFeature RSAT-AD-PowerShell
    Import-Module ActiveDirectory
    Install-ADServiceAccount gmsa_db02
    ```


### 6. Installing SQL Server

1. **Download and Install SQL Server**:
   - Download SQL Server 2019 Express and install it on both `DB01` and `DB02`.
   - Configure the Database Engine features during installation.

2. **Set gMSA as Service Account**:
   - Configure SQL Server to run under the gMSA on both `DB01` and `DB02`.
   - Follow these steps on `DB01` and `DB02`:

     1. Open **SQL Server Configuration Manager**.
     2. Navigate to **SQL Server Services**.
     3. Right-click on the **SQL Server service** (e.g., SQL Server (SQLEXPRESS)).
     4. Select **Properties** -> **Log On** tab.
     5. Select "This account", click **Browse**, and enter the gMSA account:
        - `umbrellacorp\gmsa_db01$` for `DB01`
        - `umbrellacorp\gmsa_db02$` for `DB02`
     6. Navigate to **SQL Server Network Configuration**.
     7. Under **Protocols**, enable **TCP/IP**.
     8. In the **TCP/IP** properties, go to the **IP Addresses** tab, scroll to the bottom, and set the port to `1433`.
     9. Restart the **SQL Server service**.
     10. Select **Properties** -> **Security** and change to `SQL Server and Windows Authentication mode`.

### 7. Creating and configuring database perms and accounts

   - Use the .sql scripts in below repo to create and populate the database, follow the commands in `sql account setup and db link` to create the users and assign permissions - if you change the password this will have to be reflected in `appsettings.json` when we unzip our websites.
   - [public db setup](https://github.com/kymb0/bucket/blob/main/public_research_setup.sql)
   - [secret db setup](https://github.com/kymb0/bucket/blob/main/secret_research_setup.sql)
   - [sql account setup and db link](https://github.com/kymb0/bucket/blob/main/sql_accounts_setup.md)


### 8. Create Web App

   - Install [M.NET Core Hosting Bundle](https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-8.0.6-windows-hosting-bundle-installer)
   - Download https://github.com/kymb0/bucket/blob/main/UmbrellaIntranet.zip and https://github.com/kymb0/bucket/blob/main/DevIntranet.zip and unzip to `C:\inetpub`
   - IIS Manager -> Add an Appication Pool for Intranet and Dev Intranet, slecting the corresponding unzipped folder parenting `wwwroot`, set the host field to reflect the DNS entries we setup for `intranet.umbrellacorp.local` and `dev.env.intranet.umbrellacorp.local`.

Once you unzip and host these, you should be able to navigate to `http://intranet.umbrellacorp.local`!

<img src="https://github.com/kymb0/kymb0.github.io/blob/master/assets/images/lab01/Umbrella_intranet.png" alt="Intranet" width="300"/>


## DONE

Ta Da! Now you have your very own vulnerable, repeatable, expandable lab!!

If you want to go a step further and patch this lab, my first suggestions (aside from firing the dev) would be to make `WEB01` a principal allowed to retrieve the gMSA password for `gmsa_db01` and `gmsa_db02`, install the service accounts, and change how our application connects to the DB.

### Bonus Notes! 👏

If we make these changes, what the heck is the gMSA doing under the hood on `WEB01`???

- `WEB01$` requests a Ticket Granting Ticket (TGT) from the KDC.
- KDC provides the TGT to `WEB01$`.
- SSMS running on `WEB01$` makes a request to retrieve the gMSA password.
- `WEB01$` uses the TGT to retrieve the gMSA password from the KDC.
- `WEB01$` sends an authentication request to the SQL Server on `DB01` using the gMSA credentials.
- SQL Server on `DB01` responds to the authentication request, completing the connection process.

## Attack Path (Spoiler)

<details>
  <summary>Click 4 Spoilerz</summary>

- **Initial Reconnaissance**:
  Navigate to the main intranet site. An error message reveals the local filesystem path, providing clues about the server configuration and potential points of exposure.

- **Uncovering the Development Intranet**:
  Discover a dev.env subdomain under the report issues page which, when navigated to, leads to a development version of the intranet. This site is intentionally less secure and provides a gateway to explore deeper vulnerabilities.

- **Extracting Sensitive Configuration Data**:
  Retrieve database connection string from `c:\inetpub\DevIntranet\appsettings.json` via an LFI present in the Document Management page in the Dev Intranet. This file, left insecurely accessible, demonstrates a common security oversight involving sensitive data exposure.

### Database Exploration

Utilize the exposed database credentials to connect using a legitimate account. Without escalating privileges or performing SQL injection, list linked servers and dump critical database contents. This demonstrates how legitimate access can be leveraged to achieve significant impacts, reflecting the high stakes involved even with standard user permissions.

Download `sqlcmd` from [here](https://github.com/microsoft/go-sqlcmd/releases/download/v1.8.0/sqlcmd-windows-amd64.zip).

```bash
# Connect to the database using sqlcmd
sqlcmd -S db01 -U public_db_reader -P 'threewordphrase1!' -W

# List current databases
SELECT name from sys.databases;
GO

# List tables in the 'public_research' database
select * from public_research.information_schema.tables;
GO

# Query the 'ResearchPapers' table in the 'public_research' database
select * from public_research.dbo.researchpapers;
GO

# Check for linked servers as no useful data is present
SELECT * FROM sys.servers WHERE is_linked = 1;
GO

# Query linked server databases
SELECT name FROM db02.master.sys.databases;
GO

# List tables in the _5G_enzyme_experimental database on a linked server
select * from db02._5G_enzyme_experimental.information_schema.tables;
go

# Query the ExperimentalEnzymes table on the db on the linked server
SELECT * FROM DB02._5G_enzyme_experimental.dbo.ExperimentalEnzymes;
GO
```

- Full one-liner to dump secret db: s`qlcmd -S db01 -U public_db_reader -P 'threewordphrase1!' -Q "select * from openquery(db02, 'select * from _5G_enzyme_experimental.dbo.experimentalenzymes');" -W`

