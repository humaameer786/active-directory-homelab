# Active Directory HomeLab 👩‍💻

I’m building this lab to properly learn Active Directory by actually setting up a small Windows domain environment from scratch.

The aim is to understand what is happening underneath each step, especially networking, DNS, authentication, users and groups, Group Policy, domain joins, permissions, and the kind of troubleshooting I would actually need to do in an IT support or security role.

This repo is a work in progress and will grow alongside the lab.

> I also decided to theme the lab around the TV series *The Mentalist*, which is why the users, departments and a few other names are based around the CBI.

## What I Want to Learn

By the end of this project I want to be comfortable with:

- Active Directory Domain Services
- Domain Controllers
- DNS in an Active Directory environment
- users, groups and computers
- Organizational Units
- Group Policy
- domain joins and domain logins
- password resets and account lockouts
- Windows permissions
- DHCP
- basic AD troubleshooting
- how the different parts of a Windows domain actually communicate with each other


## What Will Be in This Repo

This repo is gradually becoming a record of the lab rather than just a collection of screenshots from a finished setup.

I’m documenting:

- the overall lab design and why I chose it
- VirtualBox networking and `ADLAB-NAT`
- Windows Server 2022 installation and preparation
- Active Directory Domain Services deployment
- creation and verification of the `hoomaverse.test` forest
- DNS and how Active Directory depends on it
- Organizational Unit design
- users, groups and computer objects
- Windows client deployment and domain joining
- Group Policy
- troubleshooting notes from things that actually break
- PowerShell and Windows commands used for verification
- selected screenshots where they actually add something useful
- later Windows DHCP, file shares, permissions and IT support scenarios
- later security-focused AD exercises once the admin side is properly understood

---

# Setup

This section covers the foundation of the lab: the host machine, VM resources, VirtualBox networking, Windows Server configuration, and the decisions I made before turning the server into a Domain Controller.

## Lab Environment

### Host

- Oracle VirtualBox 7.2.6
- 16 GB RAM
- Intel Core i7-11700K
- VM storage on a separate HDD

### Server VM

- Windows Server 2022 Standard Evaluation
- Desktop Experience
- 4 GB RAM
- 2 vCPUs
- ~60 GB dynamically allocated VDI
- connected to a dedicated VirtualBox NAT Network
- hostname: `HOOMA-DC`
- IPv4: `10.10.10.10`

### Client VM

- Windows 10 Pro 22H2
- 3 GB RAM
- 2 vCPUs
- 60 GB dynamically allocated VDI
- connected to `ADLAB-NAT`
- hostname: `CLIENT01`
- IPv4: `10.10.10.20`
- DNS: `10.10.10.10`

I used Windows 10 Pro because I wanted a lightweight client that still supports a normal Active Directory domain join.

## Network Design

I created a separate VirtualBox NAT Network for the AD lab instead of bridging the VMs directly onto my normal home network.

```text
                Internet
                   |
             Windows Host
                   |
            VirtualBox NAT
              10.10.10.1
                   |
              ADLAB-NAT
             10.10.10.0/24
             /           \
        HOOMA-DC         CLIENT01
       10.10.10.10      10.10.10.20
```

Current network design:

```text
Network:          10.10.10.0/24
Subnet mask:      255.255.255.0
Gateway:          10.10.10.1

HOOMA-DC:         10.10.10.10
CLIENT01:         10.10.10.20
```

Both VMs are live on the same private lab network and can communicate with each other.

## Why I Disabled VirtualBox DHCP

I intentionally disabled the DHCP server built into the VirtualBox NAT Network.

I could have left it enabled and let VirtualBox automatically hand out IP configuration, but that would hide part of what I actually want to learn.

For now I am configuring the network manually.

Later I want Windows Server to provide DHCP itself so I can learn how DHCP works in a Windows domain environment without having VirtualBox DHCP and Windows DHCP competing on the same virtual network.

VirtualBox is basically providing the virtual network and NAT route.

Windows Server will provide the enterprise services I actually want to learn.

## Static Network Configuration

Before installing Active Directory, I gave `HOOMA-DC` a static network configuration:

```text
IP address:       10.10.10.10
Subnet mask:      255.255.255.0
Default gateway:  10.10.10.1
DNS:              1.1.1.1
```

`1.1.1.1` was temporary while the server was still just a standalone Windows Server and the AD DNS service did not exist yet.

After promotion, `HOOMA-DC` became the DNS server for `hoomaverse.test`.

The Domain Controller now uses its own DNS service locally, while domain clients point to the Domain Controller:

```text
HOOMA-DC DNS client:  127.0.0.1
CLIENT01 DNS:         10.10.10.10
```

That distinction matters because Active Directory clients need the domain DNS server to discover Domain Controllers and services such as LDAP and Kerberos.

## Network Verification

I tested the network in layers instead of opening a browser and deciding everything must be fine.

### Gateway

```powershell
ping 10.10.10.1
```

Result:

```text
Sent = 4
Received = 4
Lost = 0
```

This confirmed that `HOOMA-DC` could communicate with the VirtualBox NAT gateway.

### Internet Routing

```powershell
ping 1.1.1.1
```

Result:

```text
Sent = 4
Received = 4
Lost = 0
```

This confirmed that traffic could leave the virtual lab network.

### Public DNS

```powershell
Resolve-DnsName microsoft.com
```

The lookup successfully returned DNS records for `microsoft.com`.

So instead of just knowing that "the Internet works", I had separately verified:

```text
HOOMA-DC
    |
    v
VirtualBox Gateway   ✅
    |
    v
Internet Routing     ✅
    |
    v
DNS Resolution       ✅
```

## Server Baseline

Before installing Active Directory, I sorted out the basic server configuration so I would not be changing fundamental settings after the machine became a Domain Controller.

```text
Hostname:    HOOMA-DC
IPv4:        10.10.10.10
Gateway:     10.10.10.1
Time zone:   UTC+04:00 Abu Dhabi, Muscat
```

I renamed the server before promotion because renaming a Domain Controller later is more involved than renaming a normal Windows machine.

I also corrected the time zone before configuring the domain because Active Directory authentication, especially Kerberos, depends heavily on accurate time.

## Domain Design

For the Active Directory forest and root domain, I chose:

```text
hoomaverse.test
```

The first Domain Controller is:

```text
HOOMA-DC.hoomaverse.test
```

The NetBIOS domain name is:

```text
HOOMAVERSE
```

This is an isolated lab domain used only inside the virtual environment.

---

# Building the Domain

This is the point where the server stopped being a mostly ordinary Windows Server and became the centre of the lab.

## Installing Active Directory Domain Services

I installed the **Active Directory Domain Services (AD DS)** role on `HOOMA-DC` along with the management tools needed to administer Active Directory.

Installing AD DS does not automatically make a Windows Server a Domain Controller.

At that stage the software was installed, but there was still:

- no Active Directory forest
- no domain
- no AD database
- no domain users
- no domain authentication

The server still needed to be promoted.

## Promoting HOOMA-DC

I promoted `HOOMA-DC` to become the first Domain Controller in a completely new Active Directory forest.

The promotion configuration is:

```text
Forest:                    hoomaverse.test
Root domain:               hoomaverse.test
NetBIOS domain:            HOOMAVERSE
Forest functional level:   Windows Server 2016
Domain functional level:   Windows Server 2016
DNS Server:                Enabled
Global Catalog:            Enabled
RODC:                      Disabled
```

Because this is the first Domain Controller, the promotion created both the new forest and its root domain.

I also configured a separate **Directory Services Restore Mode (DSRM)** password.

> DSRM is used for offline Active Directory recovery if the directory services ever need to be repaired. 

Windows displayed a DNS delegation warning during promotion because there is no existing parent DNS zone for `hoomaverse.test`. That was expected here because this is a brand-new isolated forest rather than a child domain underneath existing DNS infrastructure.

## What the Domain Controller Actually Does

Before promotion, `HOOMA-DC` was basically a Windows Server with Active Directory software installed.

Promotion changed its job.

It became the central authority for `hoomaverse.test`.

The Domain Controller is now responsible for things such as:

- storing users, groups, computers and Organizational Units
- authenticating users when they log in
- authenticating computers when they join the domain
- providing DNS for the Active Directory environment
- supplying Group Policy to domain machines
- storing shared AD files and policies inside SYSVOL
- acting as a Global Catalog for directory lookups

In simple terms, instead of every Windows computer having its own completely isolated collection of users and settings, the Domain Controller gives the environment one central place to manage identity, authentication and policy.

The relationship now looks roughly like this:

```text
              hoomaverse.test
                     |
                  HOOMA-DC
                10.10.10.10
                     |
        +------------+-------------+
        |            |             |
       DNS      Authentication   Group Policy
        |            |             |
        +------------+-------------+
                     |
                  CLIENT01
                10.10.10.20
```

## DNS and the Domain Controller

Active Directory does not just happen to use DNS. It depends on it.

Domain machines use DNS to discover things like:

- which Domain Controller to contact
- where LDAP services are
- where Kerberos authentication is available
- which servers provide other AD services

That is why `CLIENT01` uses:

```text
DNS: 10.10.10.10
```

rather than pointing directly to a public DNS resolver.

A machine can have perfectly working IP connectivity and Internet access while Active Directory still fails because its DNS configuration is wrong.

That is one of the distinctions I specifically wanted this lab to make less mysterious.

## Verifying the Domain

I did not want to trust the promotion wizard alone, so I verified the domain from PowerShell after the reboot.

An Active Directory SRV lookup:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.hoomaverse.test
```

returned the Domain Controller:

```text
hooma-dc.hoomaverse.test
Port: 389
IPv4: 10.10.10.10
```

I also checked the domain and forest configuration with `Get-ADDomain` and `Get-ADForest`.

The results confirmed:

```text
Domain:           hoomaverse.test
NetBIOS:          HOOMAVERSE
Domain mode:      Windows2016Domain
Forest mode:      Windows2016Forest
Global Catalog:   HOOMA-DC.hoomaverse.test
```

Because this is currently a single-DC lab, `HOOMA-DC` also holds the FSMO roles.

At this point the domain was not just "installed". I had actually proved that AD and its DNS records were alive.

---

# Building the Active Directory Structure

Once the domain itself was fine, I moved on to creating a structure that I could actually use for users, computers, permissions and Group Policy.

## Organizational Units

Instead of dumping everything into the default Active Directory containers, I created my own top-level `hooma` OU:

```text
hoomaverse.test
└── hooma
    ├── users
    ├── workstations
    ├── groups
    └── servers
```

Under `users`, I used a CBI theme for the departments:

```text
users
├── CBI-Major-Crimes-Unit
├── CBI-Forensics
├── CBI-Internal-Affairs
├── CBI-Media-Relations
├── CBI-Management
└── CBI-Investigations
```

This was also where the difference between an **OU** and a **group** started making sense in practice.

An OU answers things like:

> Where does this object live, and where can policy be targeted?

A security group answers:

> What is this user a member of, and what should that membership allow them to access?

They look similar in ADUC when you are new to it, but they are doing completely different jobs.

## Users

I created domain users across the departmental OUs, including:

### Major Crimes

- Patrick Jane
- Teresa Lisbon
- Kimball Cho
- Wayne Rigsby
- Grace Van Pelt

### Other departments

- Brett Partridge
- J.J. LaRoche
- Brenda Shettrick
- Luther Wainwright
- Virgil Minelli
- Madeleine Hightower
- Sam Bosco

## Security Groups

I also created matching **Global Security** groups using a consistent `GG_CBI_*` naming pattern:

```text
GG_CBI_MajorCrimes
GG_CBI_Forensics
GG_CBI_InternalAffairs
GG_CBI_MediaRelations
GG_CBI_Management
GG_CBI_Investigations
```

Users were added to the relevant groups.

Later these groups will become much more useful when I start working with shared folders and permissions, because I want to grant access through groups rather than assigning permissions directly to individual users.

---

# CLIENT01

The lab stopped being a single-server exercise once I built the first Windows client.

## Building the Client

`CLIENT01` is a Windows 10 Pro 22H2 VM with:

```text
RAM:       3072 MB
vCPUs:     2
Disk:      60 GB dynamically allocated VDI
Network:   ADLAB-NAT
Hostname:  CLIENT01
```

I used a local `clientadmin` account during the initial setup.

One useful lesson here was the distinction between a local account and a domain account:

```text
CLIENT01\clientadmin
```

exists only on `CLIENT01`, while something like:

```text
HOOMAVERSE\teresa.lisbon
```

is stored and authenticated by Active Directory.

## CLIENT01 Networking

I gave the client a static address:

```text
IP address:       10.10.10.20
Subnet mask:      255.255.255.0
Default gateway:  10.10.10.1
DNS:              10.10.10.10
```

Before joining the domain, I verified that CLIENT01 could actually reach the Domain Controller:

```powershell
ping 10.10.10.10
```

Result:

```text
Sent = 4
Received = 4
Lost = 0
```

Then I verified that the client was really using `HOOMA-DC` for DNS:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4
```

The active Ethernet adapter showed:

```text
10.10.10.10
```

## Discovering Active Directory from CLIENT01

A successful ping only proves that one IP can reach another.

For a domain join, I also wanted to confirm if CLIENT01 could actually discover Active Directory through DNS.

I ran:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.hoomaverse.test
```

CLIENT01 received:

```text
NameTarget:  hooma-dc.hoomaverse.test
Port:        389
IPv4:        10.10.10.10
```

That meant CLIENT01 was not just able to reach `10.10.10.10`. It could ask DNS:

> Where is the LDAP Domain Controller for `hoomaverse.test`?

and get the correct answer.

## Joining the Domain

I joined CLIENT01 to:

```text
hoomaverse.test
```

using domain administrator credentials.

After the reboot, Active Directory created the CLIENT01 computer object. I moved it from the default `Computers` container into:

```text
hooma
└── workstations
    └── CLIENT01
```

That move becomes important for Group Policy because I can now target workstation policies at my own OU rather than leaving the client sitting in the default container forever.

## Domain Login

I then logged into CLIENT01 using a real domain user:

```text
HOOMAVERSE\teresa.lisbon
```

and verified the identity with:

```powershell
whoami
```

The result confirmed that Windows was using the HOOMAVERSE domain account rather than the local CLIENT01 account.

That completed the full chain:

```text
CLIENT01
    |
    v
AD DNS discovery
    |
    v
Domain join
    |
    v
Computer object in Active Directory
    |
    v
Domain user authentication
    |
    v
Successful domain login
```

---

# Group Policy

This is where the OU structure started doing something more interesting than looking organised in ADUC.

## First Computer-Side GPO

I created:

```text
CBI Workstation Logon Notice
```

and linked it to:

```text
hooma
└── workstations
```

Because `CLIENT01` lives inside that OU, the policy targets the workstation.

Inside the GPO I configured:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── Security Options
```

with:

```text
Interactive logon: Message title
CBI Workstation Notice

Interactive logon: Message text
This workstation is for authorized CBI personnel only.
```

After restarting CLIENT01, the message appeared before the Windows sign-in screen. 

That was the first point where Group Policy stopped being a diagram and became something I could actually see happening on the client.

## Verifying the GPO

I also verified the policy from CLIENT01 rather than relying only on the visible message.

```cmd
gpresult /r /scope computer
```

showed:

```text
Applied Group Policy Objects
----------------------------
CBI Workstation Logon Notice
Default Domain Policy
```

It also confirmed that Group Policy had been applied from:

```text
HOOMA-DC.hoomaverse.test
```

So I had three separate pieces of evidence:

```text
GPO configured on HOOMA-DC          ✅
GPO linked to the workstations OU   ✅
CLIENT01 reports the GPO applied    ✅
```

Selected evidence is stored under:

```text
screenshots/group-policy/workstation-logon-notice/
```

## User-Side Group Policy

I also created:

```text
CBI Major Crimes User Restrictions
```

and linked it to:

```text
hooma
└── users
    └── CBI-Major-Crimes-Unit
```

This policy is deliberately different from the workstation logon notice.

The logon notice uses **Computer Configuration**, so it follows the computer object inside the `workstations` OU.

The Major Crimes restriction uses **User Configuration**, so it follows the user accounts inside `CBI-Major-Crimes-Unit`, including Teresa Lisbon, Patrick Jane, Kimball Cho, Wayne Rigsby and Grace Van Pelt.

For the first user-side restriction, I enabled:

```text
Prohibit access to Control Panel and PC settings
```

under:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
```

After signing Teresa out and back in, Windows blocked access to Control Panel and Settings as expected.

I then verified the policy from CLIENT01 with:

```cmd
gpresult /r /scope user
```

and confirmed that:

```text
CBI Major Crimes User Restrictions
```

was listed under the applied user Group Policy Objects.

This gave me a clean practical example of the difference between computer-side and user-side Group Policy:

```text
Computer Configuration
        |
        v
workstations OU
        |
        v
CLIENT01
```

versus:

```text
User Configuration
        |
        v
CBI-Major-Crimes-Unit
        |
        v
Major Crimes users
```

---

# What Comes Next

## Finish User-Side Group Policy

The current task is finishing and verifying the Major Crimes user restriction, then checking the applied user policies from CLIENT01.

## DHCP

The lab currently uses manually assigned IP addresses.

Later I plan to install the Windows DHCP Server role and let Windows Server handle client addressing instead of VirtualBox.

That will let me learn:

- scopes
- leases
- exclusions
- reservations
- DHCP options
- how DHCP behaves inside a Windows domain

## Permissions and File Shares

I also want to build shared folders and practice the relationship between:

- users
- groups
- share permissions
- NTFS permissions

Rather than assigning permissions directly to random users, I want to manage access through the security groups I already created.

## IT Support Scenarios

Once the normal environment is working, I want to deliberately create problems such as:

- locked user accounts
- forgotten passwords
- disabled accounts
- incorrect DNS settings
- failed domain logins
- permission problems
- Group Policy issues
- domain-join failures

A working environment teaches me how to build Active Directory.

A broken one should teach me how to actually support it.

---

# Later: The Security Side

The first goal of this project is administration.

I want to understand normal Active Directory before jumping straight into attacking it.

Once I am comfortable with users, groups, DNS, Kerberos, permissions, Group Policy and domain administration, I want to extend the same environment into security exercises involving things such as:

- Active Directory enumeration
- privilege relationships
- BloodHound
- misconfiguration discovery
- authentication attacks
- defensive investigation
- hardening

