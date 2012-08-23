# Stackato Essentials #

My personal goal with this document is to provide a somewhat more gentle introduction to Cloud Foundry and Stackato than you might otherwise receive, as well as to shine light on components and functionality that might not be documented in sufficient detail elsewhere.

If you find any of the information here useful, you can thank me by [providing feedback](https://github.com/ttilley/StackatoEssentials/issues). *Please* feel free to fork, improve, and submit pull requests.

## Core Components ##

![General Architecture](http://docs.stackato.com/_images/stackato-architecture-diagram.png "stackato general architecture")

### Doozerd ###

> Doozer is a highly-available, completely consistent store for small amounts of extremely important data. When the data changes, it can notify connected clients immediately (no polling), making it ideal for infrequently-updated data for which clients want real-time updates.
> <https://github.com/ActiveState/doozerd>

Doozer is a [recent addition](http://www.activestate.com/blog/2012/08/doozer-distributed-configuration-used-heroku-and-stackato) to the stack that is unique to Stackato, replacing most configuration sources used by other implementations of Cloud Foundry. In standard Cloud Foundry one might SSH into the machine running a service to configure it (via YAML files), and restart said service to pick to those configuration changes. In Stackato you are able to configure the entire cluster from a single location and services react to configuration changes themselves. You are also able to query doozer for information on currently connected nodes.

### Prealloc ###

### Stager ###

### DEA ###

### Cloud Controller ###

The Cloud Controller handles all state transitions, manages users/apps/services, and provides the external REST API used by the stackato client (or VMC, should you so desire).

### Health Manager ###

### Router ###

The router handles all HTTP traffic into the cluster and maintains distributed routing state. The router responds to realtime updates from DEA nodes. Load balancing is performed when an app has multiple instances.

There are currently two implementations of the router component in Stackato (as of 2.2). In order to make use of websockets, one must use router2g rather than the default router.

### NATS ###

NATS is a lightweight cloud messaging system by Derek Collison, who was previously Chief Architect for TIBCO's messaging products.

Message passing, via NATS, is the foundation of the Cloud Foundry architecture. It is used for addressing and component discovery, command and control, heartbeats, and various other tasks.

#### Cloud Controller ####

* Publish: dea.discover
* Publish: dea.find.droplet
* Publish: dea.locate
* Publish: dea.start
* Publish: dea.stop
* Publish: dea.update
* Publish: droplet.updated
* Publish: healthmanager.health
* Publish: healthmanager.status
* Publish: router.register
* Publish: router.unregister
* Publish: vcap.cc.events
* Publish: vcap.component.announce
* Publish: vcap.stager.{queue}
* Subscribe: cloudcontroller.bulk.credentials
* Subscribe: cloudcontrollers.hm.requests
* Subscribe: dea.advertise
* Subscribe: router.start
* Subscribe: vcap.component.discover

#### DEA ####

* Publish: dea.advertise
* Publish: dea.heartbeat
* Publish: dea.start
* Publish: droplet.exited
* Publish: router.register
* Publish: router.unregister
* Publish: vcap.component.announce
* Subscribe: dea.discover
* Subscribe: dea.find.droplet
* Subscribe: dea.locate
* Subscribe: dea.status
* Subscribe: dea.stop
* Subscribe: dea.update
* Subscribe: dea.{uuid}.start
* Subscribe: droplet.status
* Subscribe: healthmanager.start
* Subscribe: router.start
* Subscribe: vcap.component.discover

#### Health Manager ####

* Publish: cloudcontrollers.hm.requests
* Publish: healthmanager.start
* Publish: vcap.component.announce
* Subscribe: dea.heartbeat
* Subscribe: droplet.exited
* Subscribe: droplet.updated
* Subscribe: healthmanager.health
* Subscribe: healthmanager.status
* Subscribe: vcap.component.discover

#### Router ####

* Publish: router.start
* Publish: vcap.component.announce
* Subscribe: router.register
* Subscribe: router.unregister
* Subscribe: vcap.component.discover

#### Stager ####

* Subscribe: vcap.stager.{queue}

### Data Services ###

All data services have three subcomponents: Node, Provisioner, and Gateway. The Node implements the actual service backend. The provisioner handles various logic around provisioning and unprovisioning a specific instance of the service. The gateway provides a REST interface to the provisioner. In practice the provisioner and gateway are usually implemented as a singular daemon.

Prior to Stackato 2.2, the filesystem service was a special exception to this rule. It did not have a gateway, and was assumed to exist on a singular node. This limitation has since been lifted.

## Application Deployment Flow ##

This is a shortened version of what happens when you deploy an application:

1. stackato push
2. framework detection
3. create app in CloudController
4. framework staging plugin
5. droplet creation
6. request DEA for app
7. an available DEA node responds
8. droplet is deployed to the DEA
9. DEA starts the app
10. upon successful start, the router creates a route

## Cluster Setup ##

### Roles ###

### Fog ###

```
require 'rubygems'
require 'fog'

STACKATO_V207_AMI       = "ami-595bf530"

AWS_ACCESS_KEY_ID       = ENV["AWS_ACCESS_KEY_ID"]
AWS_SECRET_ACCESS_KEY   = ENV["AWS_SECRET_ACCESS_KEY"]
EC2_REGION              = ENV["EC2_REGION"]             || "us-east-1"
EC2_AVAILABILITY_ZONE   = ENV["EC2_AVAILABILITY_ZONE"]  || "us-east-1d"

STACKATO_EMAIL          = ENV["STACKATO_EMAIL"]         || "stackato@stackato.local"
STACKATO_PASSWORD       = ENV["STACKATO_PASSWORD"]      || "stackato"

connection = Fog::Compute.new({
  :provider               => :AWS,
  :aws_access_key_id      => AWS_ACCESS_KEY_ID,
  :aws_secret_access_key  => AWS_SECRET_ACCESS_KEY,
  :region                 => EC2_REGION
})

server = connection.servers.bootstrap({
  :private_key_path   => "~/.ec2/default.pem", 
  :public_key_path    => "~/.ec2/default.pem.pub", 
  :availability_zone  => EC2_AVAILABILITY_ZONE, 
  :username           => "ubuntu", 
  :image_id           => STACKATO_V207_AMI,
  :flavor_id          => "m1.small",
  :bits               => 64
})

server.ssh(<<EOF
curl -k "https://api.$(hostname).local/stackato/license" \
     -d "email=#{STACKATO_EMAIL}&password=#{STACKATO_PASSWORD}&unix_password=#{STACKATO_PASSWORD}"
EOF
)
```

### Chef ###

### CloudInit ###

cloud-init is a highly configurable system for handling early initialization of a cloud instance. More information on cloud-init can be found at the [project's launchpad](https://launchpad.net/cloud-init). There is also [a screencast](http://www.youtube.com/watch?v=-zL3BdbKyGY) covering the cloud-config portion of cloud-init on the [ubuntucloud youtube channel](http://www.youtube.com/user/ubuntucloud).

#### Data Sources

The cloud-init package supports searching multiple "Data Sources" for information that can be used to configure a particular cloud instance. The list of potential sources is configured in the `datasource_list` key, with the first entry found to be valid becoming the data source for that run. This value, as well as any detailed configuration in `datasource`, should be set locally in `cloud.cfg` or injected early via some other means (like the kernel cmdline). Any Ubuntu based EC2 AMIs will contain EC2 as a data source by default, otherwise the list will contain `["NoCloud", "ConfigDrive", "OVF", "MAAS"]`.

*Note: If the `datasources_list` value from `/etc/cloud/cloud.cfg` is not being used, check for an `/etc/cloud/cloud.cfg.d/90_dpkg.cfg` and run `dpkg-reconfigure cloud-init` if found.*

Note: Selecting the "Ec2" data source may result in lengthy boot times if the EC2 metadata API is unavailable. If you firewall off the metadata API for security purposes and do not wish to use it with cloud-init, make sure to edit `datasource_list`. For a convenient method of removing access to the metadata API after cloud-init has retrieved any necessary data, enable the `disable-ec2-metadata` module.

If you are using a deployment platform that provides some level of EC2 compatibility then you may want to ensure that the metadata IP (169.254.169.254) is reachable and that the HTTP endpoints (ex: /2009-04-04/meta-data/instance-id) are available ahead of time. For additional configurability, you also have the option of manually setting `metadata_urls` on the data source:

```
datasource:
  Ec2:
    metadata_urls:
      - http://169.254.169.254:80
      - http://instance-data:8773
```

While CloudStack provides a metadata API with similarities to the one provided by EC2, there are notable differences that make it incompatible. In particular, the metadata API will be available via the default gateway and thus cannot be sanely configured ahead of time. There is a seperate "CloudStack" datasource that handles these differences.

Valid data sources:

  * **NoCloud**: reads info from `/var/lib/cloud/seed` only
  * **ConfigDrive**: reads data from OpenStack Config Drive
  * **OVF**: reads data from OVF Transports
  * **MAAS**: reads data from Ubuntu MAAS
  * **EC2**: reads data from EC2 metadata service
  * **CloudStack**: reads data from CloudStack metadata service

#### Frequency

Scripts and configuration directives may be acted upon with the following frequencies:

1. once **ever**
2. once per instance
3. once per boot (always)

You can usually assume that the default frequency for an action is once per instance. On every run, cloud-init will check what the current "instance id" is and compare this value with the previously recorded id. This makes it easy to do things like clone an existing system and ensure that a base set of actions are performed when the clone comes online. You may, for example, delete any pre-existing chef configuration and re-register as a new client.

*Warning: On EC2, a full shutdown will result in your VM receiving a new instance-id.*

The following modules, however, are loaded with a default frequency of "always":

* bootcmd
* disable-ec2-metadata
* final-message
* resizefs
* scripts-per-boot
* update_etc_hosts
* update_hostname

Note that without accompanying configuration, a module that is loaded on each boot will still have no effect on the system.

It is possible to change the run frequency of a module. Under the module section where it is defined, replace the string name with an array of `[name, frequency, arguments]`. Within this context, frequency must be named `once-per-instance`, `always`, or `once`.

#### Order of Execution

The cloud-init package is run in several stages during bootstrap:

* root filesystem is mounted
* cloud-init-local runs (**cloud-init start-local**)
  * purge any cached data and metadata
  * if a valid (non-network) data source is found then perform a normal start using that data source
* all configured network interfaces come up
* cloud-init runs (**cloud-init start**)
  * check the kernel cmdline for a `cloud-config-url` parameter. If present, fetch and save the content of this URL to `/etc/cloud/cloud.cfg.d/91_kernel_cmdline_url.cfg`
  * ensure /var/lib/cloud and subdirectories are created
  * set the current instance id
  * update cached user data and metadata
  * the `cloud_init_modules` value from cloud-config is read and modules are run in the order that they appear. these are, by default:
    1. bootcmd
    2. resizefs
    3. set_hostname
    4. update_hostname
    5. update_etc_hosts
    6. ca-certs
    7. rsyslog
    8. ssh
* any remaining filesystems will be mounted by this point
* rsyslog will be running by this point
* cloud-config runs (**cloud-init-cfg all config**)
  * the `cloud_config_modules` value from cloud-config is read and modules are run in the order that they appear. these are, by default:
    1. mounts
    2. ssh-import-id
    3. locale
    4. set-passwords
    5. grub-dpkg
    6. apt-pipelining
    7. apt-update-upgrade
    8. landscape
    9. timezone
    10. puppet
    11. chef
    12. salt-minion
    13. mcollective
    14. disable-ec2-metadata
    15. runcmd
    16. byobu
* all runlevel rc scripts run
* cloud-final runs (**cloud-init-cfg all final**)
  * the `cloud_final_modules` value from cloud-config is read and modules are run in the order that they appear. these are, by default:
    1. rightscale_userdata
    2. scripts-per-once
    3. scripts-per-boot
    4. scripts-per-instance
    5. scripts-user
    6. keys-to-console
    7. phone-home
    8. final-message

#### risks ####

## Monitoring ##

### Ganglia ###

## Auto-Scaling ##

## Troubleshooting ##

### Application Crash ###

If an application crashes, the linux container for that app sticks around for about an hour by default before getting cleaned up. This allows you to ssh into the container and perform any necessary debugging. Logs will also be available to you this way, even if they are not directly available via the dashboard any longer.

>Note: you can access an app as it's owner, or as an admin with the --group='owner@email' option to the stackato CLI. The way option ordering works, the group option must come before the ssh command:
>````
>stackato --group='owner@email.com' ssh appname
>````

In the scenario of an application crash:

1. The DEA node detects the unexpected exit and broadcasts a message to NATS
2. Routers remove the crashed app from the routing table
3. The Health Manager notifies the Cloud Controller
4. The Cloud Controller re-starts the instance

### DEA Crash ###

In the scenario of a DEA node crash:

1. Applications handled by the DEA become unavailable
2. The Health Manager notices the missing instances and notifies the Cloud Controller
3. The Cloud Controller requests DEA nodes for the apps
4. As DEA nodes reply, application instances will be started on these new DEA nodes

Note that as part of the Prealloc/DEA init that occurs when starting the DEA service, previously existing linux containers are cleaned up and removed. If you need any of that data for debugging or analysis then you will want to copy it to another location than /lxc/containers or /mnt/lxc/containers.

### Inotify ###

````
fs.inotify.max_user_instances = 4096
fs.inotify.max_user_watches = 32768
fs.inotify.max_queued_events = 65536
````

