# Broken DNS on CLIENT01

I deliberately changed CLIENT01's DNS from `10.10.10.10` to `1.1.1.1` to see what would actually break if a domain client was using the wrong DNS server.

Before changing anything I checked that:

- CLIENT01 was using `10.10.10.10` for DNS
- CLIENT01 could ping HOOMA-DC
- the AD LDAP SRV record could be resolved successfully

I then changed only the IPv4 DNS server to:

```text
1.1.1.1
```

The IP address and gateway were left on DHCP.

After changing DNS:

- CLIENT01 could still reach `10.10.10.10` by IP
- `microsoft.com` still resolved normally
- the Active Directory SRV lookup failed with `DNS name does not exist`

The failed lookup was:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.hoomaverse.test
```

This showed me that normal networking and Internet DNS could still work while Active Directory discovery was broken.

I then changed CLIENT01 back to:

```text
Obtain DNS server address automatically
```

so DHCP could give it the correct DNS server again:

```text
10.10.10.10
```

I flushed the local DNS cache with:

```cmd
ipconfig /flushdns
```

At this point I expected AD DNS to work again straight away, but the first lookup returned:

```text
This operation returned because the timeout period expired
```

So even though CLIENT01 was showing `10.10.10.10` as its DNS server again, the AD lookup was still timing out.

Instead of assuming the DNS server was broken, I checked the problem in layers.

I confirmed that:

- CLIENT01 was using `10.10.10.10` again
- HOOMA-DC still had `10.10.10.10/24`
- both VMs could reach the `10.10.10.1` VirtualBox gateway
- TCP 53 for DNS was reachable
- TCP 389 for LDAP was reachable
- TCP 445 for SMB was reachable
- CLIENT01 still showed the network as `DomainAuthenticated`

One slightly confusing part was that ping between CLIENT01 and HOOMA-DC was unreliable for a moment, even though the actual services on ports 53, 389 and 445 were reachable.

That reminded me not to rely on ping alone when deciding whether a server or service is available.

On HOOMA-DC I then checked that:

- the DNS Server service was running
- DNS was listening on both UDP and TCP port 53
- the AD SRV record resolved locally using `127.0.0.1`
- the same record resolved using `10.10.10.10`
- the Windows Firewall DNS rules were enabled
- DNS rules allowed both UDP and TCP traffic

I also tried forcing the AD lookup directly against `10.10.10.10`, including a TCP-only DNS query, and those initially timed out as well.

CLIENT01 could still successfully resolve `microsoft.com` directly through `10.10.10.10`, which showed that remote DNS queries to HOOMA-DC were working.

I then tested the AD DNS zone again from CLIENT01:

```powershell
Resolve-DnsName hooma-dc.hoomaverse.test -Server 10.10.10.10
```

```powershell
Resolve-DnsName hoomaverse.test -Type SOA -Server 10.10.10.10
```

```powershell
Resolve-DnsName _ldap._tcp.dc._msdcs.hoomaverse.test -Type SRV -Server 10.10.10.10
```

All three worked.

Finally I ran the normal lookup again without manually specifying a DNS server:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.hoomaverse.test
```

and CLIENT01 successfully found:

```text
hooma-dc.hoomaverse.test
10.10.10.10
LDAP port 389
```

So the DNS configuration was fully restored.

The main thing I took from this was that having Internet access or being able to reach another machine by IP does not mean Active Directory is working.

I also learned that a DNS timeout does not automatically mean the DNS server itself is broken. In this case I checked the service, ports, firewall rules, DNS records and actual queries before changing anything, and the issue cleared while the configuration itself remained correct.