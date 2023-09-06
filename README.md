Samba-Active-Directory Infrastructure
=====================================
=====================================

This Repository along with the following document will lead you through installing a infastructure simular to a **Active-Directory** Structure on Microsofts Windows Server on a Ubunut Linux machine.

### Prerequisites

For this infrastructure you will need a three Ubuntu machines and optionally a Windows Server if you want to control the Samba-Directory from Windows RSAT Tools.

We will basically set up a Domain-Controller, a Fileserver that uses the SRVDC01 to authenticate and two clients to act as customers.

- SRVDC01
- SRVFS01
- Client 1 (Linux)
- Client 2 (Windows)

All of the Servers need to be able to communicate to each other via IPv4.

As this is done for a school project the realm is called "sam159.iet-gibb.ch", but feel free to take whatever name you want to use.

### Configure DNS Server and Hostname (SRVDC01)

On the SRVDC01 do this in the */etc/hosts* file:

```bash
<srvdc01-ip> srvdc01.sam159.iet-gibb.ch
```
And this in the */etc/hostname* file:

```bash
srvdc01.sam159.iet-gibb.ch
```

On every other Server set the **IP** of the SRVDC01 as DNS Server and **sam159.iet-gibb.ch** as search-domain (or distribute it via dhcp service)

### Samba installation (SRVDC01)

Install required software:

```bash
sudo apt update -y && sudo apt upgrade

# If heimdal-kerberos prompts you for the domain, just enter "sam159.iet-gibb.ch". The server and the admin-server can be set to "srvdc01.sam159.iet-gibb.ch"
sudo apt install samba smbclient heimdal-clients

sudo apt install acl attr build-essential libacl1-dev libattr1-dev

sudo apt install libblkid-dev libgnutls28-dev libreadline-dev python3-dev

sudo apt install python3-dnspython gdb pkg-config libpopt-dev libldap2-dev

sudo apt install libbsd-dev krb5-user docbook-xsl libcups2-dev acl ntp ntpdate

sudo apt install net-tools git winbind libpam0g-dev dnsutils lsof
```

Save original samba conf:

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.original
```

Now use the command:
```bash
sudo samba-tool domain provision
```
To initialize the domain. This will ask you some questions:

- As Realm use **SAM159.IET-GIBB.CH**, this corresponds to the **FQDN**.
- For domain i suggest something like **SAM159**, it corresponds to the - **NetBIOS** Name.
- As Server Role select **dc**.
- And for the DNS backend select **SAMBA_INTERNAL**.
- For DNS forwarder I suggest to use a public dns like **1.1.1.1**.
- Then you will also need to set the Administrator-Password (this is like the Master-Password of the KDC)


Disable the DNS-Resolver of the machine (systemd-resolver):
```bash
sudo systemctl disable systemd-resolved && sudo systemctl stop systemd-resolved
```

Because we disabled systemd-resolved, we need to enter the nameservers into */etc/resolv.conf*:

```bash
nameserver 127.0.0.1
nameserver 1.1.1.1
```

If the */etc/resolv.conf* file still has a softlink on it, remove it and create the file. Just make sure to give it *644* rights.

Check if all settings were configured correctly in the */etc/samba/smb.conf*, it should now look simular to this:
```bash
[global]
dns forwarder = 1.1.1.1
netbios name = SRVDC01
realm = SAM159.IET-GIBB.CH
server role = active directory domain controller
workgroup = SAM159
[netlogon]
path = /var/lib/samba/sysvol/sam159.iet-gibb.ch/scripts
read only = No
[sysvol]
path = /var/lib/samba/sysvol
read only = No
```


Enable Samba systemd service:
```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
``` 

To work with the AD from the server, set this Kerberos-Client configuration */etc/krb5.conf*:
```bash
sudo rm -f /etc/krb5.conf
# First set a softlink from the samba krb5.conf
sudo ln -sf /var/lib/samba/private/krb5.conf /etc/krb5.conf
# Then add the kdc location:
[libdefaults]
    default_realm = SAM159.IET-GIBB.CH
    dns_lookup_realm = false
    dns_lookup_kdc = true
[realms]
    SAM159.IET-GIBB.CH = {
        kdc = srvdc01.sam159.iet-gibb.ch
        default_domain = sam159.iet-gibb.ch
    }
```

Now restart the server:
```bash
sudo reboot
```
#### Test Services

You can test if the setup works by checking if the samba (bzw. smbd) service is listening to following ports:

- :49152
- :49153
- :49154
- :3268
- :3269
- :389
- :135
- :139
- :464
- :53
- :88
- :636
- :445

Listening ports can be displayed with **netstart -tlpn**.

Next check with **samba_dnsupdate --verbose** for failures or errors on the DNS service.

And check the most important DNS-Names to be resolved:

```bash
host -t SRV _kerberos._tcp.sam159.iet-gibb.ch
host -t SRV _gc._tcp.sam159.iet-gibb.ch
host -t SRV _ldap._tcp.sam159.iet-gibb.ch
host -t A srvdc01.sam159.iet-gibb.ch
```

But as we all know, real men test in production.

#### Set up DNS Records

**IMPORTANT** in this section I will use the Network 192.168.110.0/24 where the SRVDC01 is 192.168.110.10. You need to specifically set these IP Addresses corresponding to your infrastructure.

Some domain services need to reverse lookup records, for this we need to create a Reverse lookup zone:
```bash
sudo samba-tools dns zonecreate 192.168.110.10 110.168.192.in-addr.arpa -Uadministrator
```

**INFORMATION**: The samba dns tool is used like this:
**Add:**
samba-tool dns add <Server> <Zone> <Name/IP> <Type> <IP-Address/Hostname> -U<Operation-User>
**Delete:**
samba-tool dns delete <Server> <Zone> <Name/IP> <Type> <IP-Address/Hostname> -U<Operation-User>
**Add Zone:**
samba-tool dns zonecreate <Server> <Zone-Name> -U<Operation-User>

Now lets directly set up the reverse lookup record for the Realm:
```bash
sudo samba-tool dns add 192.168.110.10 110.168.192.in-addr.arpa 10 PTR srvdc01.sam159.iet-gibb.ch -Uadministrator
```

For the Fileserver lets now set a basic A record:
```bash
sudo samba-tool dns add 192.168.110.10 sam159.iet-gibb.ch srvfs01 A 192.168.110.11 -Uadministrator
```

Test the client connection with smbclient:

```bash
smbclient -L localhost -Uadministrator
# With the '-k' flag you can make it authenticate via kerberos
# If you use kerberos authentication, make sure to not use localhost, as it is not a valid SPN on the kerberos-database
smbclient -L srvdc01 -k
```

If everything works, you can now add users to samba (principals) with a command like this:

```bash
sudo samba-tool user add felix.blume
```
Use this to list the users:
```bash
sudo samba-tool user list
```

Finally you can display the FSMO Rules with the command:
```bash
sudo samba-tool fsmo show
```