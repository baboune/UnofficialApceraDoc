# UnofficialApceraDoc

Instructions for setting up a BareOS Enterprise Edition of Apcera.

## Base cluster

Minimum viable install for BareOS:
- 4 servers: orchestrator, singleton, central and one instance manager (IM).<br>
- base OS: ubuntu 14.04 (minimal or server).<br>
- the IM (1 of the 4 servers) must have a primary partition with 20GB space.<br>

And with minimum viable size as per https://docs.apcera.com/installation/deploy/sizing/#minimum-viable-deployment-mvd.

Nb    | type         | RAM  | Disk   | Installed components
----- | ------------ | ---- | ------ | ------
1 | orchestrator     | 2GB |  8GB  |     orchestrator-server, orchestrator-database
1 | central          | 4GB |  20GB |     router, api-server, flex-auth-server, nats-server, <br>job-manager, health-manager, cluster-monitor, <br> metrics-manager, auditlog-database, component-database, events-server
1 | singleton        | 4GB |  20GB  |  auth-server, package-manager (local), redis-server, <br>graphite-server, statsd-server, stagehand
1 | instance-manager | 8GB |  100GB |  instance-manager

It is important to remember that the Apcera setup will change the hostnames of the initial servers.


Each Instance Manager (IM) machine host must have its disk partitioned so that the IM has a dedicated volume on which to run container (job) instances. The majority of the disk should be allocated to the IM. During the cluster.conf crafting step, you must provide a physical disk partition so that the IM can use LVM to create necessary logical partitions. See http://docs.apcera.com/installation/bareos/bareos-install-reqs/#partition-requirements.

In short, create a primary partition sufficient for the OS (say 20 GB) ("/dev/sda1"), and another partition that is not formatted for the job instances.  

For example, on a single disk machine using /dev/sda, it can look like this:
```
Disk /dev/sda: 64.4 GB, 64424509440 bytes
255 heads, 63 sectors/track, 7832 cylinders, total 125829120 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0006d8c7

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    39063551    19530752   83  Linux
/dev/sda2        39063552   125829119    43382784   83  Linux
```

Note: If you follow the Apcera documentation, during the ubuntu setup a single partition is created (e.g. /dev/sda1).  As such, after the install, one must login to the IM, and manually add a partition.  

Using the above example with a single disk:
```
$ fdisk /dev/sda
```
Then "n" to create a new partition, "p" for primary, "2" for number, accept defaults for start and end, then "w" to write changes to disk.

That partition information is required to craft the cluster.conf file.


## General recommendations

### Forget using Vagrant

The chef deployment will bind to the eth0 interface (ie first interface) in the list.  Since Vagrant sets up an additional interface (NAT) on the first interface, the orchestrator will fail to deploy on the other apcera hosts.
It gets worse based on https://docs.apcera.com/installation/bareos/bareos-install-reqs/: "Network interface requirements
Currently you must use eth0 as the name for the network interface on the Ubuntu host."

### External DNS
While the documentation refers to using a DNS, the DNS is not used during the setup.

The orchestrator-cli (using chef) sets the hostname to clustername-uuid, where the uuid is the one seen in the orchestrator ssh menu.  This makes it possible to correlate logs back to their hosts after extracting them to another system for analysis. The same chef recipe adds the local hostname to /etc/hosts on each host to allow local name resolution.  There is no interaction with dns, none should be required.  Nothing outside the host is aware of the hostname. All cluster services are configured by IP.

After the initial setup of Apcera, it is only necessary for two DNS records to be created: {DOMAIN} and *.{DOMAIN}. the cluster FQDN/wildcard entries, i.e. *.your.cluster.name, pointing to the routers or whatever load balancer you have in front of them.

### Problems with http://docs.apcera.com/installation/bareos/bareos-install-scripts/

There are two sections that are confusing and error prone in the online documentation.  I suggest not to spend too much time on those:
* Provision the Orchestrator machine host
* Provision all other cluster machine hosts

Similar steps are described below in a different manner.


## Pre-requisites

Install [Ubuntu Server 14.04](http://docs.apcera.com/installation/bareos/bareos-install-ubuntu/) on each machine host.

Review the [provisioning script](http://docs.apcera.com/installation/bareos/bareos-install-ubuntu/docs.apcera.com/installation/bareos/bareos-install-reqs/#provisioning-scripts) requirements.

Each script requires internet access as well as access to a valid DNS to resolve external hostnames like "apcera.com". Air gapped installations require running your own apt-mirror. This is not described here.

AFAIK, air gapped installations are not supported atm.

## Steps after installing ubuntu

### Run scripts
Once Ubuntu is setup on each server. There are two scripts to run as root (or sudo): 
* provision_base.sh: Apply to all servers except "orchestrator" type.
* provision_orchestrator.sh: Apply to "orchestrator" type only.

These scripts update the kernel, create required users, add repositories, set up OS level security, run Chef recipes, and install necessary agents for each cluster server. 

They also change the sshd_config to only allows the following users to SSH in:
* On orchestrator node: ops, orchestrator, and root users. 
* On othe nodes: ops, and root users. 

The provisioning scripts are available to Apcera Platform Enterprise Edition licensed customers via the Apcera Customer Support Portal. Sign into the [Support Portal](http://support.apcera.com/) with your account credentials to download the scripts. To run the scripts, see provisioning cluster hosts.

The script "provision_base.sh" must be ran on all servers except the orchestrator.

The script "provision_orchestrator.sh" must be ran on the orchestrator server only.

Note: 
* Running those scripts require all servers to have internet access.


At this point, the servers are almost "ready" to deploy apcera.

### Things to consider

#### Apcera key management

The scripts contain two ssh public keys (apcera-user-ca, apcera-special-ops) that get deployed on all machines.  Those keys are for Apcera ops only.  The private keys are owned by Apcera.

Since, as an installer, we do not have a matching private key, it can not be used to SSH into the servers.

Searching through the scripts with "ssh-rsa", one finds:

1. around line 165, provision_base.sh:
```
(
cat <<'EOP'
cert-authority ssh-rsa AAAAB...N apcera-user-ca
ssh-rsa AAAAB3N...4n/D apcera-special-ops
EOP
)> /etc/ssh/userauth/root
```

2. Around line 207, provision_base.sh:
```
 Write the SSH keys for the ops user. This includes the master Apcera ops key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
)> /etc/ssh/userauth/root
```

3. around line 165, provision_orchestrator:
```
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
) > /etc/ssh/userauth/root
```

2. Around line 207, provision_orchestrator.sh:
```
 Write the SSH keys for the ops user. This includes the master Apcera ops key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
) > /etc/ssh/userauth/root
```

It is unclear at this step which ones should a user modify, it is most likely a bug as the shell scripts basically do the same thing twice.
(??)

#### Restricted access to "ops", "root"

In http://docs.apcera.com/installation/bareos/bareos-install-reqs/#provisioning-scripts, it says: "The scripts change the sshd_config to only allows the ops, orchestrator, and root users to SSH in. You can change that to allow other users if you desire." 

The statement from the doc is partially false:
* The provision_base.sh script will limit access to "ops" and "root".
* The provision_orchestrator.sh script will limit access to "ops", "orchestrator" and "root".

If other users should be allowed access, then either modify the provision_xx.sh script or the sshd_config file:

```
  ...
  AllowUsers ops orchestrator root myuser
  ...
```

And restart ssh for changes to sshd_config to take effect: 
```
$ service restart ssh
```

### Setup ssh access

During the "provision_x.sh" phase, the scripts will modify how one accesses ssh.  Basically, the scripts change the sshd_config to only allows some users to SSH in, and they change how the identity check is performed.

One such modification in sshd_config is this:
```
AuthorizedKeysFile /etc/ssh/userauth/%u
```

As such, if one wants to allow for another user to log on the machine, then place the public keys in the matching location. E.g for user "lmcnise":
```
cat lmcnise_key.pub > /etc/ssh/userauth/lmcnise
chmod 600 /etc/ssh/userauth/lmcnise
```

Where lmcnise_key.pub is the "lmcnise" public key generated via ssh-keygen on linux.

For remote log in via SSH, you should update the scripts with your public SSH key for each user (root, ops, orchestrator).There are different options at this point to enable remote SSH and have knowledge of the key:
1.  Add a key in the provision_x.sh scripts in the proper location.
2.  Before launching the provision script create on orchestartor node one or two key and coy them on singleton, instance-manager and central
[‎2016-‎09-‎06 15:10] Andrea Severini: 
using ssh-copy-id in destination folder /etc/ssh/userauth/ops and /etc/ssh/userauth/root so copy from rochestrator to singleton, central and IM


One option is ....


### Craft the cluster.conf

The cluster.conf file uses an Apcera proprietary format called "dconf".  The format is defined here: [http://docs.apcera.com/reference/dconf/]. 

Working example of cluster.conf: [config/bareos-cluster-mine.conf](config/bareos-cluster-mine.conf).

#### Allowing remote SSH

In order to allow remote SSH, it is possible to add a public key information to the cluster.conf.

TBD.

#### Adding partition device for the instance managers (IMs)

As part of the cluster.conf, one must specify a partition to allocate for the running jobs.  The partition information is required in the following section:

```
chef: {

 [...]

 "mounts": {
    "instance-manager": {
      "device": "/dev/sda2"
    }
```

The "device" value must match the name of the partition that will be allocated for the jobs.

#### Making sure the admin can log in the Apcera console (apc)

If using a single basic-auth user "admin", that user must also be listed in the "admins" section, otherwise it won't have any access.

The initial cluster.conf looked like this:
```
 "admins": [
       "username@apcera.me"
     ],
```
Since my admin user in [config/bareos-cluster-mine.conf](config/bareos-cluster-mine.conf) is defined as:
```
"auth_server": {
  "identity": {
    "basic": {
      "enabled": true,
      "users": [
        {
          "name": "admin",
          "password": "ericsson"
        }
      ]
    },
  },
```
Which means use basic-auth, and user name is "admin".  Then the "admins" section must be:
```
 "admins": [
       "admin@apcera.me"
     ],
```

Note: The `@apcera.me` distinguishes the "basic auth" usernames from names that come from other auth providers.

# Tips and tricks

## Clean up after a failed deployment

Do a `orchestrator-cli teardown`, and then on each of the BareOS hosts run `rm /etc/chef/*` to reset their chef configuration completely.

## Orchestrator logs 

On the different hosts being setup, the logs are in /var/log/orchestrator-agent/.

The file to look at is "current":
```
$  tail -f /var/log/orchestrator-agent/current
```


# Automated setup

1. Setup

a. The Vagrant file will only setup the DNS node to use as a nameserver (192.168.50.100).
b. Manually spin up 3 VMs, Ubuntu 14 server or minimal. 
c. Define IP/Hostname such as:
192.168.50.30 orchestrator.apcera.test
192.168.50.31 central.apcera.test
192.168.50.32 singleton.apcera.test
192.168.50.33 im-01.apcera.test
And with minimum viable size as per https://docs.apcera.com/installation/deploy/sizing/#minimum-viable-deployment-mvd.
                      RAM   DISK
1   orchestrator      2GB   8GB       orchestrator-server, orchestrator-database
1   central           4GB   20GB      router, api-server, flex-auth-server, nats-server,
                                      job-manager, health-manager, cluster-monitor, metrics-manager, auditlog-database, component-database, events-server
1   singleton         4GB   20GB    auth-server, package-manager (local), redis-server, 
                                    graphite-server, statsd-server, stagehand
1   instance-manager  8GB   100GB   instance-manager


After running the provision_orchestrator.sh script, it will not be possible to login via ssh tp the orchestrator host. SSH is limited to key authentication.  

To allow orchestrator to login vis ssh, one needs to physically log onto the host, then:
- Generate an SSH key pair (if you do not already have one).
- Create a file under /etc/ssh/userauth/, 
  $ touch /etc/ssh/userauth/orchestrator
- Open file /etc/ssh/userauth/orchestrator for editing
- Copy and paste your public key at the end of the file (the entire string starting with ssh-rsa)
- Save the file



Download the release bundle.  Get rid of the proxy and accessing the internet.


Authentication:
- Basic auth should work
- Enable google-auth: https://docs.apcera.com/installation/identity/auth-google/
- cluster.conf:
"auth_server": {
      "identity": {
        "basic": {
          "enabled": true,
          "users": [
  {
    "name": "USERNAME",
    "password": "PASSWORD"
  }
]


In the cluster.conf:

# Add any SSH keys that should be placed on machines provisioned within the
    # cluster. Each key should be a string entry in the array. The SSH keys will
    # be placed on the host by Chef and allow the system to be accessible by the
    # "ops" user, or using the "orchestrator-cli ssh" command. Any changes will
    # be applied during initial step of the Deploy action.
    "ssh": {
      "custom_keys":[
      # Name and contact in for this key here
       "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9Czz98kASpEpaz2KjsLK3QHgRnAP8QtxIVjqo2bNclxefmMDrxqpquznCldALoJfNcMxEHLnsTbOXBdOFknVGzv4UDXzUJX8sAQK1nmeT2TjNjbbvXx0Nn0CsDi7HAbLMYIJ8lrmq/c4s0G/Ws+0OX3NgqKvhYjdJDClqVh6YNIIsgv7MdE4yoKwiHebqbZJy61X2zQiwWX4J8e3b4nmvzONhkDkrAwJ/+ywuHcxOiv+xSaiZsdRuP7ixZ5sDRImYlMtGg1pv+Ujeazs005eBAgOI1kRAWaMJTZ9Uo+AurKt82uijfMULK492cKSJs+9mdASYbfql8OV4W6S6tDCH orchestrator@bare04"
      ]
    },
    

Replace with own public key to enable ssh.

https://downloads.apcera.com/continuum/bundles/release-2.2.0.tar.gz

Add flag to release bundle:
orchestrator-cli deploy -c cluster.conf --release-bundle ....tar.gz

      
# Links
* http://docs.apcera.com/installation/bareos/bareos-install-config/
* https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
* https://docs.apcera.com/installation/bareos/bareos-install-config/
* https://docs.apcera.com/installation/deploy/orchestrator/#copy-clusterconf
* https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
* http://docs.apcera.com/installation/bareos/bareos-install-reqs/
