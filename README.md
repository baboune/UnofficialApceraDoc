# UnofficialApceraDoc


Minimum viable install for BareOS:
- 4 servers  <br>
- install ubuntu 14.04 (minimal or server) <br>
- 1 of the 4 servers is the IM and must have a primary partition with 20GB space <br>

Then run provision_base.sh script on all.

Then run provision_orchestrator.sh script on orchestrator server only.

Craft the cluster.conf.

Working example of cluster.conf [config/bareos-cluster-mine.conf]

###########3

1. Setup
Forget using Vagrant, the chef deployment will bind to the eth0 interface (ie first interface) in the list.  Since Vagrant sets up an additional interface (NAT) on the first interface, the orchestrator will fail to deploy on the other apcera hosts.
It gets worse: https://docs.apcera.com/installation/bareos/bareos-install-reqs/, "Network interface requirements
Currently you must use eth0 as the name for the network interface on the Ubuntu host."

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


---------------
https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation

4 servers, install ubuntu
1 the IM has a 20GB partition

Then run provision_base.sh script on all.
Then run provision_orchestrator.sh script on all.

Craft the cluster.conf.



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




DNS:
- Define a wildcard for the base domain.
- 


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

Verify the partition before the cluster.conf is used for provisioning, chek secrtion: 
"instance-manager": {
        "device": "/dev/sda4"
      }

https://downloads.apcera.com/continuum/bundles/release-2.2.0.tar.gz

Add flag to release bundle:
orchestrator-cli deploy -c cluster.conf --release-bundle ....tar.gz

      
Lync:
http://docs.apcera.com/installation/bareos/bareos-install-config/
https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
https://docs.apcera.com/installation/bareos/bareos-install-config/
https://docs.apcera.com/installation/deploy/orchestrator/#copy-clusterconf
https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
http://docs.apcera.com/installation/bareos/bareos-install-reqs/

SSH:
also, in http://docs.apcera.com/installation/bareos/bareos-install-reqs/#provisioning-scripts, it says: "The scripts change the sshd_config to only allows the ops, orchestrator, and root users to SSH in. You can change that to allow other users if you desire." but the scripts as downloaded from https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation  contains: "AllowUsers ops root"- So orchestrator can not login via ssh. Bug? Or documentation pb?
add: 
    AllowUsers ops orchestrator root




 chef sets the hostname to clustername-uuid, where the uuid is the one seen in the orchestrator ssh menu.  This makes it
 possible to correlate logs back to their hosts after extracting them to another system for analysis.  The same chef recipe
 adds the local hostname to /etc/hosts on each host to allow local name resolution.  There is no interaction with dns, none
 should be required.  Nothing outside the host is aware of the hostname. All cluster services are configured by IP.
    