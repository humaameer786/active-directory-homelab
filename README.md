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
- PowerShell and Windows commands used for verification
- selected screenshots where they add something useful
- Windows DHCP
- Windows file sharing and group-based NTFS permissions
- later IT support scenarios
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
- IPv4: `10.10.10.100` through a Windows DHCP reservation
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
       10.10.10.10      10.10.10.100
```

Current network design:

```text
Network:          10.10.10.0/24
Subnet mask:      255.255.255.0
Gateway:          10.10.10.1

HOOMA-DC:         10.10.10.10 (static)
CLIENT01:         10.10.10.100 (DHCP reservation)
```

Both VMs are live on the same private lab network and can communicate with each other.

## Why I Disabled VirtualBox DHCP

I intentionally disabled the DHCP server built into the VirtualBox NAT Network.

I could have left it enabled and let VirtualBox automatically hand out IP configuration, but that would hide part of what I actually wanted to learn.

I started the lab with manual addressing and later installed the Windows DHCP Server role on `HOOMA-DC`.

That means the responsibilities are now split like this:

```text
VirtualBox
└── provides the virtual network and NAT gateway

HOOMA-DC
├── Active Directory
├── DNS
└── DHCP
```

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
                10.10.10.100
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

These groups became especially useful when I moved into file sharing and permissions. Instead of assigning access directly to individual users, I used the security groups to control which departments could access particular resources.

That gave the groups a practical purpose beyond simply organising users inside Active Directory.

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

During the original CLIENT01 build, I deliberately gave the client a static address:

```text
IP address:       10.10.10.20
Subnet mask:      255.255.255.0
Default gateway:  10.10.10.1
DNS:              10.10.10.10
```

Using a static address at this stage kept the networking predictable while I was joining the machine to the domain and testing Active Directory.

Later in the lab I replaced this manual configuration with Windows DHCP and gave CLIENT01 a DHCP reservation for `10.10.10.100`.

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

At this stage CLIENT01 was still using its original static address of `10.10.10.20`. The later move to DHCP did not change the DNS design: CLIENT01 still uses `10.10.10.10` as its DNS server because domain clients need to query the Active Directory DNS service running on HOOMA-DC.

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

# Windows DHCP

I originally kept CLIENT01 on a static address because I wanted the basic networking to be completely predictable while I was building the domain.

Once AD, DNS and Group Policy were working, I finally went back to the decision I made at the start of the lab and let Windows Server take over DHCP.

## Installing and Authorizing DHCP

I installed the DHCP Server role on HOOMA-DC and completed the post-install configuration so the server was authorized in Active Directory.

That authorization step matters in a domain environment because a Windows DHCP server should not just appear on the network and start handing out configuration without being trusted by AD.

## ADLAB Client Scope

I created an IPv4 scope called:

ADLAB Client Scope

with this address pool:

```text
Start:        10.10.10.100
End:          10.10.10.199
Subnet mask:  255.255.255.0
Prefix:       /24
Lease:        8 days
```

I did not need exclusions inside the pool because the important infrastructure addresses are already outside it:

```text
10.10.10.1   VirtualBox gateway
10.10.10.10  HOOMA-DC
```

## DHCP Scope Options

The scope options are:

```text
003 Router           10.10.10.1
006 DNS Servers      10.10.10.10
015 DNS Domain Name  hoomaverse.test
```

The DNS option is especially important. A domain client getting an address from DHCP still needs to use HOOMA-DC for DNS so it can discover the Active Directory services for hoomaverse.test.

## Moving CLIENT01 from Static to DHCP

On CLIENT01 I changed IPv4 from the original manual configuration to:

Obtain an IP address automatically
Obtain DNS server address automatically

The client then received:

```text
IPv4 address:    10.10.10.100
Subnet mask:     255.255.255.0
Default gateway: 10.10.10.1
DHCP server:     10.10.10.10
DNS server:      10.10.10.10
DNS suffix:      hoomaverse.test
```

I verified that from CLIENT01 with:

```cmd
ipconfig /all
```

and then checked Address Leases on HOOMA-DC, where the same CLIENT01 lease appeared from the server side.

So the DHCP path was working end to end rather than just looking correct in the wizard.

## DORA

This was also a good point to put the classic DHCP DORA process into something real:

```text
Discover
   ↓
Offer
   ↓
Request
   ↓
Acknowledge
```

CLIENT01 broadcasts that it needs network configuration, HOOMA-DC offers an available lease, CLIENT01 requests it, and HOOMA-DC acknowledges the lease along with the gateway, DNS and domain options.

## CLIENT01 Reservation

After confirming the normal dynamic lease worked, I added CLIENT01 to Reservations so it keeps 10.10.10.100.

That means the client is still configured for DHCP and still asks the server for its network settings, but the DHCP server recognises CLIENT01 and gives it the same address each time.

That is different from manually typing 10.10.10.100 into Windows.

Static address
= configured manually on the client

DHCP reservation
= client uses DHCP, server reserves a specific address for it

HOOMA-DC itself stays statically configured at 10.10.10.10 because the Domain Controller, DNS server and DHCP server should remain reliably reachable.

Selected DHCP evidence is stored under:

```text
screenshots/dhcp/
```

---

# File Sharing and Permissions

After setting up users and security groups earlier in the lab, I wanted to use them for something more practical than simply organising Active Directory objects.

I created a departmental SMB share for the CBI Major Crimes team and used the existing `GG_CBI_MajorCrimes` security group to control access.

## Creating the Major Crimes Share

On `HOOMA-DC`, I created:

```text
C:\Shares\CBI-MajorCrimes
```

and shared it over SMB as:

```text
\\HOOMA-DC\CBI-MajorCrimes
```

This introduced another useful distinction:

```text
Local folder path
C:\Shares\CBI-MajorCrimes

        vs

Network share path
\\HOOMA-DC\CBI-MajorCrimes
```

The first refers to the folder directly on the server.

The second is the UNC path clients use to access the same resource over the network.

## NTFS Permissions

I disabled inherited permissions on the Major Crimes folder so I could control the ACL explicitly.

The final NTFS permissions were:

```text
SYSTEM                  Full Control
Administrators          Full Control
GG_CBI_MajorCrimes      Modify
```

The `GG_CBI_MajorCrimes` group received **Modify** rather than Full Control.

That allows members to:

```text
read files
create files
edit files
delete files
list folder contents
```

without giving them permission to take ownership of the folder or rewrite its security configuration.

I also removed broad inherited user access so membership of the Major Crimes security group became the deciding factor.

## Share Permissions

At the SMB share layer, I configured:

```text
Authenticated Users    Change
Administrators         Full Control
```

The share permissions are deliberately broader than the NTFS permissions.

This let me see how the two permission layers work together:

```text
Share permission
        +
NTFS permission
        =
Effective network access
```

An authenticated domain user can reach the SMB share layer, but they still need the correct NTFS permissions on the underlying folder.

## Testing Authorized Access

I signed into CLIENT01 as:

```text
HOOMAVERSE\teresa.lisbon
```

Teresa is a member of:

```text
GG_CBI_MajorCrimes
```

I opened:

```text
\\HOOMA-DC\CBI-MajorCrimes
```

and confirmed that Teresa could access the share and create and edit files.

The access path was therefore:

```text
Teresa Lisbon
      |
      v
Authenticated domain user
      |
      v
Share permission passes
      |
      v
Member of GG_CBI_MajorCrimes
      |
      v
NTFS Modify permission
      |
      v
Access allowed ✅
```

## Testing Unauthorized Access

I then signed into CLIENT01 as Luther Wainwright, a user from the Management department.

Luther is a valid authenticated domain user but is not a member of:

```text
GG_CBI_MajorCrimes
```

When I tried to open:

```text
\\HOOMA-DC\CBI-MajorCrimes
```

Windows denied access.

That produced the opposite path:

```text
Luther Wainwright
      |
      v
Authenticated domain user
      |
      v
Share permission passes
      |
      v
Not a member of GG_CBI_MajorCrimes
      |
      v
No matching NTFS permission
      |
      v
Access denied ❌
```

This was useful because it proved the permissions from both directions rather than simply confirming that one authorized account could open the folder.

At this point I had successfully used an Active Directory security group to control access to a Windows file share and verified both authorized and unauthorized access from a domain-joined client.

Selected evidence for this part of the lab is stored under:

```text
screenshots/file-sharing/
```

---

# What Comes Next

## IT Support Scenarios

Now that the normal environment is working, I want to start deliberately creating problems and troubleshooting them rather than continuing to add more services just for the sake of it.

The next exercises will include scenarios such as:

- locked user accounts
- forgotten and expired passwords
- disabled accounts
- incorrect DNS settings
- failed domain logins
- permission problems
- Group Policy issues
- domain-join failures

The goal is to practise identifying which layer is actually failing before changing anything.

A working environment has taught me how to build Active Directory.

Breaking parts of it deliberately should teach me how to support it.

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

