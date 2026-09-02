# Active Directory HomeLab 👩‍💻

I’m building this lab to properly learn Active Directory by actually setting up a small Windows domain environment from scratch. The aim is to understand what is happening underneath each step, especially networking, DNS, authentication, users and groups, Group Policy, domain joins, and the kind of troubleshooting I would actually need to do in an IT support or security role.

This repo is a work in progress and I’ll keep updating it as I build the lab.

## What I Want to Learn

By the end of this lab I want to be comfortable with:

- Active Directory Domain Services
- Domain Controllers
- DNS in an Active Directory environment
- Users, groups and computers
- Organizational Units
- Group Policy
- Domain joins and domain logins
- Password resets and account lockouts
- Windows permissions
- DHCP
- Basic AD troubleshooting
- How the different parts of a Windows domain actually communicate with each other

Later I also want to use the environment for security-focused AD exercises.

## What will be in this repo

This repo will gradually build into a full record of the lab, not just the final working setup.

I’m planning to document:

- the overall lab design and why I chose this setup
- VirtualBox networking and the `ADLAB-NAT` configuration
- Windows Server 2022 installation and baseline setup
- Active Directory Domain Services installation
- creation of the `hoomaverse.test` forest
- DNS configuration and how AD uses DNS
- Organizational Unit design
- users, groups and computer objects
- Windows client deployment and domain joining
- Group Policy configuration
- Windows DHCP
- file shares and NTFS permissions
- common IT support scenarios like password resets, lockouts and login issues
- troubleshooting notes from anything that breaks along the way
- PowerShell used for verification and AD administration
- network and AD structure diagrams
- selected screenshots that actually prove something useful
- later security-focused AD exercises once the admin side is properly understood

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
- Connected to a dedicated VirtualBox NAT Network

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
    Windows Server     Windows Client
     10.10.10.10        10.10.10.20
```

Current network plan:

```text
Network:          10.10.10.0/24
Subnet mask:      255.255.255.0
Gateway:          10.10.10.1

Windows Server:   10.10.10.10
Windows Client:   10.10.10.20
```

## Why I Disabled VirtualBox DHCP

I intentionally disabled the DHCP server built into the VirtualBox NAT Network.

I could have left it enabled and let VirtualBox automatically give the VMs their IP configuration, but that would hide part of what I actually want to learn.

For now I am configuring the network manually.

Later I want Windows Server to provide DHCP itself so I can learn how DHCP works in a Windows domain environment without having two different DHCP servers competing on the same virtual network.

VirtualBox is basically providing the virtual network and NAT route.

Windows Server will provide the services I actually want to learn.

## Static Network Configuration

The Windows Server currently uses:

```text
IP address:       10.10.10.10
Subnet mask:      255.255.255.0
Default gateway:  10.10.10.1
DNS:              1.1.1.1
```

`1.1.1.1` is only being used temporarily.

Once Active Directory and the DNS Server role are configured, the server will use the AD DNS service instead.

Eventually the domain clients should use:

```text
DNS: 10.10.10.10
```

because Active Directory relies heavily on DNS to locate Domain Controllers and other domain services.

## Network Verification

I tested the network in layers instead of just opening a browser and assuming everything worked.

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

This confirmed that the server could communicate with the VirtualBox NAT gateway.

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

This confirmed that traffic could leave the lab network through VirtualBox NAT.

### DNS

```powershell
Resolve-DnsName microsoft.com
```

The lookup successfully returned DNS records for `microsoft.com`.

So at this point I had separately verified:

```text
Windows Server
      |
      v
VirtualBox Gateway   ✅
      |
      v
Internet             ✅
      |
      v
DNS Resolution       ✅
```

## Planned Domain

The current plan is to create a new Active Directory forest using:

```text
adlab.test
```

This will be an isolated lab domain used only inside the virtual environment.


## Why I’m Documenting This

I don’t want this repo to just be a list of screenshots showing that I clicked through some Windows menus.

I want to document the important decisions, what each component is doing, how I verified that it works, and anything that breaks along the way.

The point of the project is not just to end up with a working Domain Controller.

It is to understand why it works.