# Active Directory HomeLab 👩‍💻

I’m building this lab to properly learn Active Directory by actually setting up a small Windows domain environment from scratch.

The aim is to understand what is happening underneath each step, especially networking, DNS, authentication, users and groups, Group Policy, domain joins, permissions, and the kind of troubleshooting I would actually need to do in an IT support or security role.

This repo is a work in progress and will grow alongside the lab.

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

Once I understand the administration side properly, I also want to use the same environment for security-focused Active Directory exercises.

## What Will Be in This Repo

This repo will gradually become a record of the lab rather than just a collection of screenshots from a finished setup.

I’m planning to document:

- the overall lab design and why I chose it
- VirtualBox networking and `ADLAB-NAT`
- Windows Server 2022 installation and preparation
- Active Directory Domain Services deployment
- creation of the `hoomaverse.test` forest
- DNS and how Active Directory depends on it
- Organizational Unit design
- users, groups and computer objects
- Windows client deployment and domain joining
- Group Policy
- Windows DHCP
- file shares and NTFS permissions
- common IT support scenarios
- troubleshooting notes from things that actually break
- PowerShell used for verification and administration
- network and Active Directory diagrams
- selected screenshots where they actually add something useful
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

A Windows client VM will be added later and joined to the domain.

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

`CLIENT01` is part of the design but has not been built yet.

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

I used `1.1.1.1` temporarily so the server could resolve public DNS names before the AD DNS service existed.

Once `HOOMA-DC` becomes the Domain Controller and DNS server, the lab will use the Domain Controller for DNS instead:

```text
DNS: 10.10.10.10
```

Domain clients will also use `HOOMA-DC` for DNS because Active Directory relies heavily on DNS to locate Domain Controllers and other domain services.

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

### DNS

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

That distinction will matter later when I deliberately start breaking things.

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

The first Domain Controller will be:

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

This is the point where the server stops being a mostly ordinary Windows Server and starts becoming the centre of the lab.

## Installing Active Directory Domain Services

I installed the **Active Directory Domain Services (AD DS)** role on `HOOMA-DC` along with the management tools needed to administer Active Directory.

Installing AD DS does not automatically make a Windows Server a Domain Controller.

At this stage the software was installed, but there was still:

- no Active Directory forest
- no domain
- no AD database
- no domain users
- no domain authentication

The server still needed to be promoted.

## Promoting HOOMA-DC

I configured `HOOMA-DC` to become the first Domain Controller in a completely new Active Directory forest.

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

Because this is the first Domain Controller, the promotion will create both the new forest and its root domain.

I also configured a separate **Directory Services Restore Mode (DSRM)** password.

DSRM is used for offline Active Directory recovery if the directory services ever need to be repaired. The password itself is obviously not stored anywhere in this repo.

The prerequisites check completed successfully before starting the promotion.

Windows also displayed a DNS delegation warning because there is no existing parent DNS zone for `hoomaverse.test`. That is expected here because this is a brand-new isolated forest rather than a child domain being added underneath existing DNS infrastructure.

## What the Domain Controller Actually Does

Before promotion, `HOOMA-DC` is basically a Windows Server with Active Directory software installed.

Promotion changes its job.

It becomes the central authority for `hoomaverse.test`.

The Domain Controller will be responsible for things such as:

- storing users, groups, computers and Organizational Units
- authenticating users when they log in
- authenticating computers when they join the domain
- providing DNS for the Active Directory environment
- supplying Group Policy to domain machines
- storing shared AD files and policies inside SYSVOL
- acting as a Global Catalog for directory lookups

In simple terms, instead of every Windows computer having its own completely isolated collection of users and settings, the Domain Controller gives the environment one central place to manage identity, authentication and policy.

Eventually the relationship will look something like this:

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

That is why the future client will use:

```text
DNS: 10.10.10.10
```

rather than pointing directly to a public DNS resolver.

A machine can have perfectly working IP connectivity and Internet access while Active Directory still fails because its DNS configuration is wrong.

That is one of the distinctions I specifically wanted this lab to make less mysterious.

## Verifying the Domain

Once the promotion and automatic reboot are complete, I will verify the resulting Active Directory environment instead of assuming that a successful wizard means everything works.

This section will be updated with the actual verification checks and results once `hoomaverse.test` is live.

---

# What Comes After the Domain Exists

Once the first Domain Controller is running properly, the next stages of the lab will move beyond infrastructure setup.

## CLIENT01

I will build a Windows client on the same `ADLAB-NAT` network, configure it to use `HOOMA-DC` for DNS, and join it to:

```text
hoomaverse.test
```

This will let me start testing actual domain authentication instead of doing everything locally on the server.

## Active Directory Structure

Rather than dumping every account into the default containers and calling it a day, I want to design a small but sensible Active Directory structure using:

- Organizational Units
- users
- security groups
- computer objects

The final structure will be documented once I have designed and built it.

## Group Policy

Once there are domain users and a domain-joined client, I want to use Group Policy to centrally control Windows settings and see how policies are applied, inherited and troubleshot.

## DHCP

The lab currently uses manually assigned IP addresses.

Later I plan to install the Windows DHCP Server role and let Windows Server handle client addressing instead of VirtualBox.

That will also let me learn scopes, leases, exclusions and DHCP behaviour inside a domain environment.

## Permissions and File Shares

I also want to build shared folders and practice the relationship between:

- users
- groups
- share permissions
- NTFS permissions

Rather than assigning permissions directly to random users, I want to practice managing access through groups.

## Troubleshooting

I do not want this lab to stay perfectly healthy forever.

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

For now, I’m building the castle before I start checking which windows someone forgot to lock.