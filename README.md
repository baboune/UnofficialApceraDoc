Disclaimer: December 10th 2016, I am no longer using Apcera. 

# UnofficialApceraDoc

Instructions for setting up a BareOS Enterprise Edition of Apcera.

These instructions/comments are collected while experimenting with such a 4 servers BareOS setup.

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

The `orchestrator-cli` (using chef) sets the hostname to clustername-uuid, where the uuid is the one seen in the orchestrator ssh menu.  This makes it possible to correlate logs back to their hosts after extracting them to another system for analysis. The same chef recipe adds the local hostname to /etc/hosts on each host to allow local name resolution.  There is no interaction with dns, none should be required.  Nothing outside the host is aware of the hostname. All cluster services are configured by IP.

After the initial setup of Apcera, it is only necessary for two DNS records to be created: {DOMAIN} and *.{DOMAIN}. the cluster FQDN/wildcard entries, i.e. *.your.cluster.name, pointing to the routers or whatever load balancer you have in front of them.

Note: Un my deployment, the *.{DOMAIN} is set to apcera.test and points to the host in the cluster running the "router" component in the cluster.conf ie the central (192.168.50.31). Using dnsmasq, /etc/dnsmasq.conf:

```
address=/#.apcera.test/192.168.50.31
address=/.apcera.test/192.168.50.31

```

### Problems with http://docs.apcera.com/installation/bareos/bareos-install-scripts/

There are two sections that are confusing and error prone in the online documentation.  I suggest not to spend too much time on those:
* [Provision the Orchestrator machine host](http://docs.apcera.com/installation/bareos/bareos-install-scripts/#provision-the-orchestrator-machine-host)
* [Provision all other cluster machine hosts](http://docs.apcera.com/installation/bareos/bareos-install-scripts/#provision-all-other-cluster-machine-hosts)

Similar steps are described below using a different approach.


## Pre-requisites

Install [Ubuntu Server 14.04](http://docs.apcera.com/installation/bareos/bareos-install-ubuntu/) on each machine host.

Review the [provisioning script](http://docs.apcera.com/installation/bareos/bareos-install-ubuntu/docs.apcera.com/installation/bareos/bareos-install-reqs/#provisioning-scripts) requirements.

Each script requires internet access as well as access to a valid DNS to resolve external hostnames like "apcera.com". Air gapped installations require running your own apt-mirror. This is not described here.

AFAIK, air gapped installations are not supported atm.

## Steps after installing ubuntu

### Run scripts
Once Ubuntu is setup on each server. There are two scripts to run as `root` (or sudo): 
* `provision_base.sh`: Apply to all servers except `orchestrator` type.
* `provision_orchestrator.sh`: Apply to `orchestrator` type only.

These scripts update the kernel, create required users, add repositories, set up OS level security, run `Chef` recipes, and install necessary agents for each cluster server. 

They also change the sshd_config to only allows the following users to SSH in:
* On orchestrator node: `ops`, `orchestrator`, and `root` users. 
* On other nodes: ops, and `root` users. 

The provisioning scripts are available to Apcera Platform Enterprise Edition licensed customers via the Apcera Customer Support Portal. Sign into the [Support Portal](http://support.apcera.com/) with your account credentials to download the scripts. To run the scripts, see provisioning cluster hosts.

The script `provision_base.sh` must be ran on all servers except the orchestrator.

The script `provision_orchestrator` must be ran on the orchestrator server only.

Note: 
* Running those scripts require all servers to have internet access.

At this point, the servers are almost "ready" to deploy apcera.

### Things to consider

#### Restricted access to `ops`, `root`

In http://docs.apcera.com/installation/bareos/bareos-install-reqs/#provisioning-scripts, it says: "The scripts change the sshd_config to only allows the `ops`, `orchestrator`, and `root` users to SSH in. You can change that to allow other users if you desire." 

The statement from the doc is partially false:
* The `provision_base.sh` script will limit access to `ops` and `root`.
* The `provision_orchestrator` script will limit access to `ops`, `orchestrator` and `root`.

If other users should be allowed access, then either modify the `provision_xx.sh` script or the sshd_config file:

```
  ...
  AllowUsers ops orchestrator root myuser
  ...
```

And restart ssh for changes to sshd_config to take effect: 
```
$ service restart ssh
```

#### Apcera key management

The scripts contain two ssh public keys (apcera-user-ca, apcera-special-ops) that get deployed on all machines.  Those keys are for Apcera operations people only.  The private keys are owned by Apcera and not distributed.

Since, as an installer, we do not have a matching private key, it can not be used to SSH into the servers.

Searching through the scripts with "ssh-rsa", one finds two locations per script for public SSH keys.

#### `provision_base.sh`

1. Around line 165, setup `root` user:
```
# Write the SSH keys for the root user. This includes the master Apcera root key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAAAB...N apcera-user-ca
ssh-rsa AAAAB3N...4n/D apcera-special-ops
EOP
)> /etc/ssh/userauth/root
```

2. Around line 207, setup `ops` user:
```
# Write the SSH keys for the ops user. This includes the master Apcera ops key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
) > /etc/ssh/userauth/ops
```

#### `provision_orchestrator`

1. Around line 165, setup `root` user:
```
# Write the SSH keys for the root user. This includes the master Apcera root key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
) > /etc/ssh/userauth/root
```

2. Around line 207, setup `ops` user:
```
# Write the SSH keys for the ops user. This includes the master Apcera ops key,
# as well as the user certificate authority key.
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
EOP
) > /etc/ssh/userauth/ops
```

#### Adding one's SSH key

Let's assume that you have generated an SSH key. That key has a private and a public part: mykey.pub and mykey.pem.

Only the cluster.conf entry is mandatory to ensure remote SSH access.  Once a "orchestrator deploy" occurs `Chef` will be  managing the key list.  As such, adding keys in the `provision_xx.sh` scripts is just to ensure that you retain access to the host prior to the deploy, in case something goes wrong.

It is unclear at this time which user is used by `orchestrator-cli`. So, the easiest approach is most likely to add the public key part (mykey.pub) to both sections i.e. for both the `root` and `ops` sections in the `provision_xx.sh` scripts.  

Correction: Jeremy Strout said it is the `ops` user that is required by `orchestrator-cli`.

Optional: Adding the key to a `provision_xx.sh` script, e.g. add mykey.pub for `ops`:
```
(
cat <<'EOP'
cert-authority ssh-rsa AAA...5N apcera-user-ca
ssh-rsa AAA...4n/D apcera-special-ops
ssh-rsa AAV...45n/D mykey
EOP
) > /etc/ssh/userauth/ops
```
(Copy and paste content of mykey.pub.)

This will guarantee that the key is distributed to all servers in the cluster for both `root` and `ops` during bootstraping.

Mandatory: Add SSH key to "cluster.conf" file.
```
chef: {
  "continuum": {
    
    [...]

    "ssh": {
      "custom_keys":[
       "ssh-rsa AAAAB3Nz...W6S6tDCH orchestrator@bare04",
       # The below is the mykey.pub to allow mykey.pem to be loaded in ssh-agent and `orchestrator-cli` to work
       "ssh-rsa AAAAB...22O/CN1Lxz"
      ]
    },
```
This guarantees the SSH key is managed by Chef.

Now, there is another interesting aspect of the `orchestrator-cli` in that it requires the matching private key to be within the user session context (ssh-agent). In other words, if one does not add his own key to the `cluster.conf`/`provision_xx.sh` files, then one can not use commands like:
```
$ orchestrator-cli collect
$ orchestrator-cli ssh
```
Those commands will fail by requesting a password.

The reason is that above `orchestrator-cli` commands wont work without a primary key in the ssh-agent, and since the apcera-special-ops and apcera-user-ca are not available to us, then one MUST include a public key in the `cluster.conf`/`provision_x.sh` with a known private key AND that key must be loaded in the ssh-agent.


### Craft the cluster.conf

The cluster.conf file uses an Apcera proprietary format called `dconf`.  The format is defined [here - dconf](http://docs.apcera.com/reference/dconf/). 

In the section above, I describe how to add the public key (mykey.pub) to the cluster.conf file. This is necessary to allow remote SSH within the cluster nodes from the orchestrator.

There is a lot of information about cluster.conf [here](http://docs.apcera.com/installation/config/cluster-conf/)

Working example of cluster.conf: [config/bareos-cluster-mine.conf](config/bareos-cluster-mine.conf).

#### Adding partition device for the instance managers (IMs)

As part of the cluster.conf, one must specify a partition to allocate for the running apcera jobs (containers, capsules, etc.).  The partition information is required, and can be found in the following section:

```
chef: {

 [...]

 "mounts": {
    "instance-manager": {
      "device": "/dev/sda2"
    }
```

The "device" value must match the name of the created partition that is allocated for the apcera jobs.  In my case, it is "/dev/sda2"

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

## Deploy

To speed up deployment, it is recommended to download a release from the support portal.  This will prevent various round-trips between the orchestrator server and the internet.

In my case, I downloaded [release-2.2.0.tar.gz](https://downloads.apcera.com/continuum/bundles/release-2.2.0.tar.gz).

```
$ orchestrator-cli deploy  -c bareos-cluster-mine.conf --release-bundle release-2.2.0.tar.gz
```

## Connect

### Using apc console

Get apc from http://docs.apcera.com/quickstart/installing-apc/.

Initially thought apce would use the name of my cluster plus the domain (ie mine.apcera.test) but it uses only the domain.

```
$ ./apc target http://apcera.test
Note: connections over HTTP are insecure and may leak secure information.

Targeted [http://apcera.test]
$ ./apc login --basic

Connecting to http://apcera.test...

Username: admin
Password: ********
```

### Using the web console

After making sure that the browser uses the proper DNS, head to http://mine.apcera.test where "mine" is the cluster name, and "apcera.test" the domain.

## Create your first Apcera job

At this point, I felt confident that it was possible to run a job.  Through the web console, and following the [Creating a job from a Docker image ](http://docs.apcera.com/quickstart/console_tasks/#createfromdocker) tutorial, it is expected to be able to select a few containers from pictures. There were none.

So, still through the console, "Home -> Add an app from Docker", let's try the "Hello World" container from https://hub.docker.com/_/hello-world/, and enter "hello-world".
```
Error - No packages satisfy dependency "os.linux"
```
Hummm...  A container image contains both the OS and execution parts in the binary.

Then I tried "Launch -> Capsule" as per [Creatomg a capsule from Ubuntu](http://docs.apcera.com/quickstart/console_tasks/#createcapsule).  No Ubuntu icon. In the "Or select a package" drop-down menu, there is "package::/apcera::console-lucid".  OK, select that, accept default, enable egress -> "Submit" -> "Capsule created". 

Success! Let's ssh.
```
$ apc capsule connect console-lucid-1

```
And it eventually times out.  Go back to web console, check logs:
```
"[system-error] "job::/sandbox/admin::console-lucid-1" now considered flapping, having failed 3 times within 5m0s"
```
Apparently something is wrong.

On the IM, let's check logs:
```
$ tail -f  /var/log//var/log/continuum-instance-manager/current
[DEBUG 2016-09-08 10:01:39.302617226 +0200 CEST pid=5549 requestid='' source='env.go:89' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting binding configuration.
[INFO  2016-09-08 10:01:39.302759473 +0200 CEST pid=5549 requestid='' source='templates.go:149' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Evaluating template with source: "/var/lib/continuum/package_cache/sha256-2ed5d5593b71b6d014aa6302e6895b7073de22fbc26f93dbc32083e20ebe0e65/app/public/index.html", info {/var/lib/continuum/instances/961de1a6/root/app/public/index.html {{ }}}
[INFO  2016-09-08 10:01:39.303070639 +0200 CEST pid=5549 requestid='' source='templates.go:149' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Evaluating template with source: "/var/lib/continuum/package_cache/sha256-2ed5d5593b71b6d014aa6302e6895b7073de22fbc26f93dbc32083e20ebe0e65/app/public/config.js", info {/var/lib/continuum/instances/961de1a6/root/app/public/config.js {{ }}}
[DEBUG 2016-09-08 10:01:39.303594326 +0200 CEST pid=5549 requestid='' source='states.go:1473' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Started setting hostname...
[DEBUG 2016-09-08 10:01:39.303857875 +0200 CEST pid=5549 requestid='' source='states.go:1485' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Completed setting hostname to "ip-169-254-0-1".
[DEBUG 2016-09-08 10:01:39.303870507 +0200 CEST pid=5549 requestid='' source='states.go:1420' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting sshd configuration for heavy.
[DEBUG 2016-09-08 10:01:39.303876358 +0200 CEST pid=5549 requestid='' source='env.go:32' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting to generate the common environment variables
[DEBUG 2016-09-08 10:01:39.303917652 +0200 CEST pid=5549 requestid='' source='env.go:47' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Setting Env variables for "init"
[DEBUG 2016-09-08 10:01:39.307633501 +0200 CEST pid=5549 requestid='' source='routes.go:266' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Verifying connectivity to 192.168.50.33:3888
[TRACE 2016-09-08 10:01:39.408423989 +0200 CEST pid=5549 requestid='' source='routes.go:282' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Successfully connected to 192.168.50.33:3888
[DEBUG 2016-09-08 10:01:39.408531872 +0200 CEST pid=5549 requestid='' source='states.go:1455' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting chrooting the initd process.
[DEBUG 2016-09-08 10:01:39.410973947 +0200 CEST pid=5549 requestid='' source='states.go:1461' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Completed chrooting to "/var/lib/continuum/instances/961de1a6/root".
[DEBUG 2016-09-08 10:01:39.410995955 +0200 CEST pid=5549 requestid='' source='env.go:32' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting to generate the common environment variables
[DEBUG 2016-09-08 10:01:39.411052830 +0200 CEST pid=5549 requestid='' source='env.go:47' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Setting Env variables for "init"
[DEBUG 2016-09-08 10:01:39.411066309 +0200 CEST pid=5549 requestid='' source='heavy.go:41' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Starting initd within the container.
[ERROR 2016-09-08 10:01:39.413562062 +0200 CEST pid=5549 requestid='' source='statemachine.go:1002' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Error setting up the container: 'Empty response.', unwinding the container to clean up the problem.
[INFO  2016-09-08 10:01:39.413648542 +0200 CEST pid=5549 requestid='' source='statemachine.go:967' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Switching state: STARTING -> STOPPING_WAIT
[TRACE 2016-09-08 10:01:39.413664372 +0200 CEST pid=5549 requestid='' source='statemachine.go:978' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Sending out state change notfication for 961de1a6-c8dc-4c12-9c67-53e08199af2c.
[INFO  2016-09-08 10:01:39.414220742 +0200 CEST pid=5549 requestid='' source='statemachine.go:967' job='daf304a7-4180-4e63-9ec0-fa9edb6ff153' instance='961de1a6-c8dc-4c12-9c67-53e08199af2c'] Switching state: STOPPING_WAIT -> STOPPING

```
The job is failing but without much indications about the actual problem.

Then following [Deploying a Static Web Site](http://docs.apcera.com/tutorials/staticsite/):
```
"[staging] Validating an index.htm or index.html file exists
[staging] Failed to update package dependencies: Failed to resolve packages: no packages satisfy dependency "package.nginx"
[staging] 
[staging] "package::/sandbox/admin::mywebsite" has failed to stage"
```

In the support section "Known Issues & notifications", there is an article about [adding packages to your Apcera Platform Enterprise Edition Environment](https://support.apcera.com/hc/en-us/articles/212072106-Adding-Packages-to-your-Apcera-Platform-Enterprise-Edition-Environment).  (Note that the link being on the support site, it most likely requires a license or EE account.)

At this point, it seems that a BareOS deployment of Enterprise Edition comes with none of the necessary packages for running any applications.  Remember that Apcera only "supports" running the Docker container image, but uses its own container technology.  This explains why none of the above worked.

In short, the link in the support points to a [JSON file](https://apcera-imports.s3.amazonaws.com/apcera_packages_ee_20160706100936.json) and a github project [packages-script](https://github.com/apcera/package-scripts).

#### Github [packages-script](https://github.com/apcera/package-scripts).

This Github project has on the September 8, 2016 5 stars and 18 contributors. 0 issues.

At this point, it does not seem possible to build the [os](https://github.com/apcera/package-scripts/tree/master/os) packages. See [135](https://github.com/apcera/package-scripts/issues/135).  

And all over packages will have dependencies on those.

#### The JSON file

Interestingly the JSON file [apcera_packages_ee_20160706100936.json](./default-packages/apcera_packages_ee_20160706100936.json) contains a series of names and URLs.

```
  ...

  {
    "Name": "ubuntu-14.04-apc3.cntmp",
    "URL": "https://apcera-imports.s3.amazonaws.com/ubuntu-14.04-apc3.cntmp",
    "Hash": "052abdd5683035b63c85361ffc18e34de93bb4a70b543a7f0aec5205059285c6",
    "Size": 70451876
  },
  {
    "Name": "ubuntu-14.04-build-essential-apc4.cntmp",
    "URL": "https://apcera-imports.s3.amazonaws.com/ubuntu-14.04-build-essential-apc4.cntmp",
    "Hash": "493abdf0642cb69c42058d55473986325dc1129e03084812e7ac5ae041ae09b7",
    "Size": 120236597
  },

  ...

```
No "os.linux" though which seemed to be the base requirement for running a Docker container image.

However, when one adds: 
```
$ wget https://apcera-imports.s3.amazonaws.com/ubuntu-14.04-apc3.cntmp
$ shasum -a 256 ubuntu-14.04-apc3.cntmp
052abdd5683035b63c85361ffc18e34de93bb4a70b543a7f0aec5205059285c6  ubuntu-14.04-apc3.cntmp
$ apc import ubuntu-14.04-apc3.cntmp
Scanning file...
Processing package "ubuntu-14.04-apc3"...
  Uploading... done!
Success!
```
This works! 

Let's also add nginx:
```
$ wget https://apcera-imports.s3.amazonaws.com/nginx-1.11.1.cntmp
$ apc import nginx-1.11.1.cntmp
```

### Apcera Hello World
After adding both "ubuntu-14.04-apc3.cntmp" and "nginx-1.11.1.cntmp", then it is possible to run an Apcera hello world job.

Following indications in [./tutorials/mywebsite/README.md](./tutorials/mywebsite/README.md) or [Deploying a Static Web Site](http://docs.apcera.com/tutorials/staticsite/):
```
$ apc app create mywebsite --start
Deploy path [/local/git/github/UnofficialApceraDoc/tutorials/mywebsite]: 
Instances [1]: 
Memory [256MB]: 
╭───────────────────────────────────────────────────────────────────────────────╮
│                             Application Settings                              │
├───────────────────┬───────────────────────────────────────────────────────────┤
│              FQN: │ job::/sandbox/admin::mywebsite                            │
│        Directory: │ /local/git/github/UnofficialApceraDoc/tutorials/mywebsite │
│        Instances: │ 1                                                         │
│          Restart: │ always                                                    │
│ Staging Pipeline: │ (will be auto-detected)                                   │
│              CPU: │ 0ms/s (uncapped)                                          │
│           Memory: │ 256MB                                                     │
│             Disk: │ 1024MB                                                    │
│           NetMin: │ 5Mbps                                                     │
│           Netmax: │ 0Mbps (uncapped)                                          │
│         Route(s): │ auto                                                      │
│  Startup Timeout: │ 30 (seconds)                                              │
│     Stop Timeout: │ 5 (seconds)                                               │
╰───────────────────┴───────────────────────────────────────────────────────────╯

Is this correct? [Y/n]: 
Packaging... done
Creating package "mywebsite"... done
Uploading package contents... done!
[staging] Subscribing to the staging process...
[staging] Beginning staging with 'stagpipe::/apcera::static-site' pipeline, 1 stager(s) defined.
[staging] Launching instance of stager 'static-site'...
[staging] Downloading package for processing...
[staging] Validating an index.htm or index.html file exists
[staging] Staging is complete.
Creating app "mywebsite"... done
Start Command: ./start_nginx
Waiting for the application to start...
App should be accessible at "http://mywebsite-h3f435.apcera.test"
Success!
```

And opening the web browser to http://mywebsite-h3f435.apcera.test, prints out "Hello Apcera" as expected.

### Some explanations

Looking at [Working with Packages](http://docs.apcera.com/packages/using/), it seems every binary has both *depends* and *provides* within its metadata. 

If the *depends* are not met by a matching provides somewhere within the Apcera list of available packages then the deployment of the job fails with an error.  

It turns out that the "ubuntu-14.04-apc3.cntmp" package that is installed previously provides the following (web console -> Packages -> ubuntu-14.04-apc3.cntmp):
* "os.linux"
* "os.ubuntu"
* "os.linux-14.04"
* "os.linux-14.04-apc3"

Which meets the requirements of nginx (web console -> Packages -> nginx-1.11.1.cntmp):
* "os.ubuntu"

At this point it is important to remember that Apcera (previously known as Continuum) is built as a replacement for Cloud Foundry.  This approach to building containers is quite the opposite of the Docker approach based on a Dockerfile i.e. explicitly specifying all the dependencies and packaging all dependencies at once in a single image.  Apcera will build the container based on its own *packages* i.e. whatever meets the dependencies.  (Whatever may not be the right term as it depends on policies.)

In a way, the above steps with installing the packages from the JSON file is a lucky strike. But it works!

Question: Why did it require a package for running a Docker image?
Apcera uses a short helper job to download the image. That short lived job runs on an IM and depends on a base linux package.  

## Tips and tricks

### Clean up after a failed deployment

Do a `orchestrator-cli teardown`, and then on each of the BareOS hosts run `rm /etc/chef/*` to reset their chef configuration completely.

## Orchestrator logs 

On the different hosts being setup, the logs are in /var/log/orchestrator-agent/.

The file to look at is "current":
```
$  tail -f /var/log/orchestrator-agent/current
```

### orchestrator-cli asks for password

In order for the `orchestrator-cli` to work properly on commands that require a remote SSH to servers, a valid private key must be loaded within the shell/session context.

This is done by the following set of actions:
```
# 1. start ssh-agent
$ eval $(ssh-agent)
Agent pid 9473
# 2. check there is no key
$ ssh-add -l
The agent has no identities.
# 3. add the key
$ ssh-add simple.pem 
Identity added: simple.pem (simple.pem)
```

### Adding a new user, and its SSH key

During the `provision_x.sh` phase, the scripts will modify how one accesses ssh.  Basically, the scripts change the sshd_config to only allows some users to SSH in, and they change how the identity check is performed.

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

### Authentication:
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



## Automated setup (TBD)

The Vagrant file will only setup the DNS node to use as a nameserver (192.168.50.100).

Manually spin up 3 VMs, Ubuntu 14 server or minimal. 

Define IP/Hostname such as:
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



      
# Links
* http://docs.apcera.com/installation/bareos/bareos-install-config/
* https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
* https://docs.apcera.com/installation/bareos/bareos-install-config/
* https://docs.apcera.com/installation/deploy/orchestrator/#copy-clusterconf
* https://support.apcera.com/hc/en-us/articles/209596146-Bare-OS-Installation
* http://docs.apcera.com/installation/bareos/bareos-install-reqs/
