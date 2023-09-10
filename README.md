Samba-Active-Directory Infrastructure
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
\
\
\
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
\
\
\
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
\
\
\
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
\
\
\
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
\
\
\
### Setup LDAP Account Manager (LAM)

The LDAP Account Manager is a really ugly webinterface that basically provides a user-friendly interface to work with LDAP.

To setup LAM first install the ldap tools with:
```bash
sudo apt install smbldap-tools && sudo apt install ldap-account-manager
```
I would also recommend ldap-utils for testing and debugging the ldap interface:
```bash
sudo apt install ldap-utils
```

As Samba as well as LAM is officially retarded they still think unencrypted LDAP by default is a good idea... Thats why we need to adjust the configuration to use LDAP+SSL (StartTLS) instead (LDAP on top of TLS).

**INFORMATION**: StartTLS is a technique that connects the client over the unencrypted LDAP (389) and then upgrades the connection to run on top of TLS. This is really flexible as it allows you to run the connection over the LDAP Port (389) while not blocking the unencrypted LDAP portbinding.
\
\
\
#### Setup LDAP+SSL on Samba Server

To setup LDAP with StartTLS on the Samba service we need to have a SSL certificate, this can be done in three ways, using a Self-Signed Certificate (comes with limitations), using a Lets-Encrypt certificate or if you're really whealty, you can also buy a official signed certificate.

I will show you how to do it with Lets-Encrypt and with a Self-Signed certificate, if you use a officially signed certificate, you just need to add it to the *smb.conf*. 

Keep in mind that you need to be the owner of the public domain if you use Lets-Encrypt, as I'm not the owner of *iet-gibb.ch*, I will use the way of the Self-Signed certificate in this tutorial, when using Lets-Encrypt it will be way easier and you can ommit some steps later on.
\
\
\
##### Lets-Encrypt Certificate

First we need to install certbot to generate the certificates:

```bash
sudo apt install certbot
```

Now you can obtain a certificate from the certbot-server and verify it with a DNS-Challange (the command may vary from DNS-Provider to DNS-Provider, the example here uses Cloudflare):

```bash
sudo certbot certonly --manual --preferred-challenges=dns -d srvdc01.sam159.iet-gibb.ch
```

Certbot will give you specific instructions on what DNS record to create.

The certbot should now have created following directory: */etc/letsencrypt/live/sam159.iet-gibb.ch/* containing the privatekey and the certificate. You can now update the Samba config */etc/samba/smb.conf* like this:

```bash
[global]
    ldap ssl = start tls
    tls keyfile  = /etc/letsencrypt/live/sam159.iet-gibb.ch/privkey.pem
    tls certfile = /etc/letsencrypt/live/sam159.iet-gibb.ch/fullchain.pem
```
As this is a globally trusted certificate authority you don't need to specify a cafile.

The certificate is valid for 90 days, you can renew it by using following command:

```bash
sudo certbot renew --quiet
```

Now restart the active-directory and verify the state:

```bash
sudo systemctl restart samba-ad-dc && sudo systemctl status samba-ad-dc
```

If you previously installed the *ldap-utils* you can now check the connections with:

```bash
# The ZZ Flag tells the ldap client to use StartTLS
ldapsearch -H ldap://srvdc01.sam159.iet-gibb.ch -x -ZZ
```
\
\
\
##### Self Signed Certificate

If you don't use a certifications server I would recommend to store the certificates in a location like this:

```bash
sudo mkdir -p /etc/ssl/samba/private
sudo mkdir /etc/ssl/samba/certs
```

Now generate the private & public key

```bash
# Generate Private key
sudo openssl genpkey -algorithm RSA -out /etc/ssl/samba/private/ldap.pem
# Generate Public certificate, make sure that the Common Name matches the Domain-FQDN (srvdc01.sam159.iet-gibb.ch)
sudo openssl req -new -x509 -key /etc/ssl/samba/private/ldap.pem -out /etc/ssl/samba/certs/ldap.cert
```

Finally make sure that you set permissions to protect the generated keys (especially the keyfile)

```bash
# You only need to do this if the owner is not already set to the executor of the samba-ad-dc
sudo chown root -R /etc/ssl/samba/private
sudo chown root -R /etc/ssl/samba/certs

# Read/Write only for owner
sudo chmod 600 -R /etc/ssl/samba/private
# Read/Write for Owner and Read for other users
sudo chmod 755 -R /etc/ssl/samba/certs
```

If you've done this, now you can add following lines to the */etc/samba/smb.conf* configuration (in the [global] section):

```bash
[global]
    ldap ssl = start tls
    tls keyfile  = /etc/ssl/samba/private/ldap.pem
    tls certfile = /etc/ssl/samba/certs/ldap.cert
    tls cafile   = /etc/ssl/samba/certs/ldap.cert
```
If you use an external ca-server, you will need to set the location of the certificate-authority file in the *cafile* option.

Now restart the active-directory and verify the state:

```bash
sudo systemctl restart samba-ad-dc && sudo systemctl status samba-ad-dc
```

If you previously installed the *ldap-utils* you can now check the connections with:

```bash
# As we have no globally signed ca certificate, we need to specify it manually here
export LDAPTLS_CACERT=/etc/ssl/samba/certs/ldap.cert
# The ZZ Flag tells the ldap client to use StartTLS
ldapsearch -H ldap://srvdc01.sam159.iet-gibb.ch -x -ZZ
```
Remember, if you use a self signed certificate authority (ca certificate) you will always need to pass this certificate authority to the LDAP-Client (or disable the check of the ca what is not recommended). If you actually plan to do this, I would recommend to create a certificate-server where clients can read the ca certificate from a NFS/SFTP Share.
\
\
\
#### Setup LDAP+SSL on LAM

Now lets configure the LAM service to use the LDAP via StartTLS.

For this you can go to the **LAM configuration** on the LAM Webinterface (*192.168.110.10/lam*). In the **LAM configuration** you can go to **Edit server profiles** and log in with the default password **lam**.
Then change the **Server settings** to something like this:

![LAM Configuration](/assets/lamconf01.png)

Enable **Activate TLS** and set the **Server address** to the ldap path of the server.


If you don't have a globally trusted certificate, we need to specify the location of the certification authority on the server.

For this do following steps:

```bash
# LAM will look into the ldap.conf file for the location of the CACERT
sudo tee /etc/ldap.conf <<EOF
TLS_CACERT /etc/ssl/samba/certs/ldap.cert
EOF

# On the LAM website it's recommended to also link the file like that:
sudo ln -f /etc/ldap.conf /etc/ldap/ldap.conf
# As some LAM versions also look for the config inside the /etc/ldap directory

# Now set Read+Execute rights for the LDAP conf as apache2 doesn't run with root privileges.
sudo chmod 755 /etc/ldap.conf
sudo chmod 755 -R /etc/ldap

sudo systemctl restart apache2
```
\
\
\
#### Setup LAM Profile

On the LDAP-Account-Manager you can now setup a LAM-Profile for a domain (in our case *sam159.iet-gibb.ch*).

A **Server Profile** can be created by navigating to **LAM configuration**->**Edit server profiles**->**Manage server profiles** and adding a profile.

I don't document the management of these LAM-Profiles here, as the GUI of LAM is really self-describing and the layout could potentially change in the future.

Following options need to be configured as shown below:
- **Profile management** -> **Template**          = **windows_samba4**
- **General settings**   -> **Server address**    = **ldap://srvdc01.sam159.iet-gibb.ch**
- **General settings**   -> **Activate TLS**      = **yes**
- **General settings**   -> **Tree suffix**       = **dc=sam159,dc=iet-gibb,dc=ch**
- **General settings**   -> **List of valid user**= **cn=Administrator,cn=users,dc=sam159dc=iet-gibb,dc=ch**
- **General settings**   -> **List of valid user**= **administrator@sam159.iet-gibb.ch**
- **Account types**      -> **LDAP suffix (Users)**       = **dc=sam159,dc=iet-gibb,dc=ch**
- **Account types**      -> **LDAP suffix (Groups)**       = **dc=sam159,dc=iet-gibb,dc=ch**
- **Account types**      -> **LDAP suffix (Hosts)**       = **CN=Computers,dc=sam159,dc=iet-gibb,dc=ch**
- **Module settings**    -> **Domains**           = **sam159.iet-gibb.ch**


After creating the LAM-Profile, you can select it on the Log-In page under **Server profile** and log in with the Domain-Administrator.

If your Domain-Administrator login doesn't work, it's likely that the password of the Domain-Administrator is to weak.

You can change it on the server with this command:
```bash
sudo samba-tool user password -Uadministrator
```
Or alternative (not recommended) you can also set following option in the [global] section of the */etc/samba/smb.conf*:

```bash
ldap server require strong auth = no
```

Another issue that I've experienced is that the format: **cn=Administrator,cn=users,dc=sam159dc=iet-gibb,dc=ch** in the **List of valid user** is not working on some LAM versions, to solve it just use the user in this format: **Administrator@sam159.iet-gibb.ch**.


After logging into the LAM interface, you can test the system by creating a user and then fetching a tgt on the server.

For this, first create a user in the LDAP Account Manager.

![LAM User Configuration](/assets/lamconf02.png)
Remember to set the password for the user!

To get the tgt on the server (you can also do it from a client):
```bash
kinit -r SAM159.IET-GIBB.CH -p felix.blume && klist
```

**INFORMATION**: The DN attribute in the LDAP Account Manager is the DistinguishedName and refers to full LDAP "path" of the user.
Example: The user "*felix.blume*" in the OU "*users*" in the domain "*sam159.iet-gibb.ch*" has following DN: 
"*cn=felix blume,cn=users,dc=sam159,dc=iet-gibb,dc=ch*"
You can get the DistinguishedName,SID and other user related information like this:
```bash
samba-tool user show felix.blume
```
\
\
\
### Set up fileservice (SRVFS01)


We now want to set up SRVFS01. To do this, you will need an Ubuntu server configured to use the search domain and DNS server that we set up earlier.

In the commands/examples I will use following configuration:
- Server-IP: 192.168.110.20
- DNS: 192.168.110.10
- Search-Domain: sam159.iet-gibb.ch


First you need to install the samba tools aswell as the kerberos heimdal client:

```bash
# As realm you need to set sam159.iet-gibb.ch
sudo apt install samba samba-common-bin smbclient heimdal-clients libpam-heimdal
```

**INFORMATION**: Because Linux ext4 filesystem uses UID (User IDs) and GID (Group IDs) while NTFS has a format called SID (Security IDs), Samba needs a tool like *winbind* that translates various things like the mentioned SID to UID/GID or the NTFS permissions into ext4 acl-controls vice-versa.

Then we also need the winbind-tools to be installed:

```bash
sudo apt install libnss-winbind libpam-winbind
```

To make samba use the winbind translator, you need to adjust the */etc/samba/smb.conf* with following attributes under the [global] section:

```bash
[global]
  workgroup = sam159
  realm = SAM159.IET-GIBB.CH
  security = ADS
  winbind use default domain = yes
  winbind refresh tickets = yes
  template shell = /bin/bash
  idmap config * : range = 10000 - 19999
  idmap config SAM159 : backend = rid
  idmap config SAM159 : range = 1000000 - 1999999
  inherit acls = yes
  store dos attributes = yes
  client ipc signing = auto
  vfs objects = acl_xattr

  ldap ssl = start tls
  tls cafile = /etc/ssl/samba/certs/ldap.cert
```

# TODO:  Set ldap certificate and test encrypted ldap

Check the configuration with:

```bash
testparm
```

**INFORMATION**: The parameters have following functions
- *workgroup*: NetBIOS name
- *realm*: fqdn realm name
- *security*: specifys whether the server serves as domain-component
- *winbind use default domain*: Sets the default domain, with this users can log in without explicit specify the domain.
- *winbind refresh tickets*: Makes winbind automatically refresh kerberos tickets if they are about to expire
- *template shell*: Sets the default shell for domain-users (with this you can e.g. log into a machine with ssh)
- *idmap config* *: These options specify the mapping between SIDs and UIDs/GIDs
- *inherit acls*: Defines whether permission inheritance is enabled
- *store dos attributes*: Enables the usage of DOS attributes like read-only, hidden, etc.
- *client ipc signing*: Defines if IPC messages are signed with a cryptographic signature every time.
- *vfs objects*: Sets the VFS module to use for using ACLs, in our case the *acl_xattr* allows Samba to store NTFS-style ACLs in extended attributes.

If the settings are set, you need to join the SRVFS01 to the domain, for this make sure that the */etc/krb5.conf* file is setup the same as it is on the domain-controller.

```bash
sudo net ads join -Uadministrator
# There will may be a error that the dns update failed, this is because we already set the dns-record for this server
```

**INFORMATION**: Linux has a NSS (Name Service Switch) service. If a programm like samba wants to aquire an information like a users password, a group or a hostname, it asks the NSS where to look this up, the NSS then determines the order in which databases are queried.
For example if a network-name needs to be resolved, NSS determines whether the dns or the hosts file is first queried.

In the NSS configuration (*/etc/nsswitch.conf*) you need to say that as third fallback it will go to winbind:

```bash
passwd: compat systemd winbind
group: compat systemd winbind
shadow: compat winbind
```

Afterwards restart the related services:

```bash
sudo systemctl restart smbd nmbd winbind
```

Now you should be able to switch the user on the SRVFS01 like this:

```bash
su - administrator@M159.IET-GIBB.CH
# We can even ommit the Domain as we set the smb.conf to use a default domain
su - administrator
# Because we set the default shell to bash it will now open a bash shell session
```


This flowdiagramm shows the process of switching the user, hardly simplified:

![SU Flowdiagramm](/assets/su_flow.png)

(Not that this is not correct flowdiagramm syntax, it's just a super simple example)

With the *id* command you can see the translated UID of the user, you should then see that the UID is inside the previously defined range: idmap config SAM159 : range = 1000000 - 1999999

As you can see the SID of the Users created with LAM or the RSAT Tools are in this format like this: *S-1-5-21-1234567890-1234567890-1234567890-1001* 
\
Winbind then looks at the RID:
\
*1001*
\
And translates this to the range defined:
\
*1000001*
\
We defined this range, because just using the RID would not work as these numbers are already in use by the users on the UNIX system like the user you are currently working with.
\
\
\
#### Transfer Samba configuration to Registry

This step is optional, I highly recommend doing it, especially if the samba-config is changed often.

There are various reasons why a file server should always be managed via the registry and not via smb.conf: 

- Clients initiate their own smbd processes, reading the entire smb.conf on connection. Modifying smb.conf makes every client reload the entire file, causing increased network traffic, especially with many clients or large configurations.
- Using the registry only transfers the changes, reducing data overhead. 
- Unlike smb.conf, an ASCII file editable by one admin at a time, the registry is a concurrent-accessible database. 
- Instead of SSH-ing to the server to edit smb.conf, the registry allows remote configuration through Windows' Regedit, even facilitating new share creation.


For the Samba-Configuration we will only need the *HKLM* hive, you can enumerate a element like this:

```bash
sudo net registry enumerate HKLM\\software
```

You can also access the registry over remote procedure calls:

```bash
# The k flag is used to authenticate with kerberos
sudo net rpc registry enumerate HKLM\\software -k -S srvfs01.sam159.iet-gibb.ch
```

**INFORMATION**: The registry in the samba environment is stored in */var/lib/samba/registry.tdb* and only accessible by *root*.


Now you can now import the local */etc/samba/smb.conf* configuration into the Registry like this:

```bash
sudo net conf import /etc/samba/smb.conf && sudo net conf list
```

After the config is imported, you can replace all options from the */etc/samba/smb.conf* that are contained in the [global] section with this:

```bash
[global]
    config backend = registry
```

Now all samba services will load the config from the registry. You can check this by executing *testparm*. The *testparm* command should, among other things, output something like this
*lp_load_ex: Switch to backend registration configuration*
\
\
\
#### Setup Samba Shares



### Setup Windows client for endusers

### Setup Linux client for endusers

### Control Active-Directory with RSAT-Tools from Windows

If you 








