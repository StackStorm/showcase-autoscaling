Cloud Autoscaling Workflow
=========

This project highlights the use of StackStorm in a generic, deployable, Autoscale Solution. Using
this pipeline, you should be able to use StackStorm to manage and grow any supported cloud, incuding
(but not limited to...)

- AWS
- DigitalOcean
- LibCloud supported platforms
- RackSpace

This base demo is setup using:

* New Relic for APM
* Chef for server provisioning
* RackSpace Cloud

# Getting Started

1. Configure all the integrations.
  - You will need to gather API keys and apply them to the file `hieradata/role/st2express`. Replace any dummy value with real API keys.
2. `vagrant up`
  - Start up the StackStorm server. This should take a moment to provision, but it will download and setup StackStorm and all supporting packs/workflows.
3. Configure upstream
  - New Relic in particular requires an API endpoint to be setup. If you provision in Vagrant locally, you may need to setup `ngrok` or ensure the NR hook makes it to your machine. The endpoint should be: http://<hostname>:10001/st2/nrhook
4. Start Playing!

NOTE: In the event you see Puppet provisioner warnings, simply run `vagrant provision`. There is an occasional
race condition within Puppet we are tracking down that is simply solved with another run.

To become familiar with how StackStorm works, we recommend that you take a look at our Getting Started video at
http://stackstorm.com/start-now/

## But I don't want to use Vagrant!

Totally fine! This pack is also configured with a masterless-puppet install to bootstrap the StackStorm server
on any platform you so choose. To install this on an arbitrary server, simply do the following....

1. Clone this repository to your server in the `/opt/puppet` directory.
  - `git clone https://github.com/stackstorm-showcase/autoscaling.git /opt/puppet`
2. Make sure all the pre-requsites are installed.
  - `/opt/puppet/script/bootstrap-puppet`
  - `/opt/puppet/script/bootstrap-linux`
3. Configure upstream
  - New Relic in particular requires an API endpoint to be setup. If you provision in Vagrant locally, you may need to setup `ngrok` or ensure the NR hook makes it to your machine. The endpoint should be: http://<hostname>:10001/st2/nrhook
4. Run Puppet
  - `role=st2express /opt/puppet/script/puppet-apply`
5. Start Playing!

NOTE: In the event you see Puppet provisioner warnings, simply run `/usr/bin/puprun` to run Puppet again.
There is an occasional race condition within Puppet we are tracking down that is simply solved with another run.

# Requirements:

* Vagrant - [version 1.7.2 or higher](https://www.vagrantup.com/downloads.html). (See https://github.com/mitchellh/vagrant/issues/3769)

## Project Goals

The goal of this project is to have a workspace that allows you to develop infrastructure in conjuction
with StackStorm, or even work on the StackStorm product itself. This project also serves as a template
to be used to begin building out and deploying infrastructure using your favorite Configuration Management
tool in conjunction with StackStorm

Currently, the project has support for the following Config Management Tools:

* Puppet

Additional workrooms will be created for the following languages:

* Chef
* Ansible
* Salt

## Usage
### Vagrant Stacks

Underpinning Vagrant is something referred to as Vagrant Stacks. This pattern is another attempt by the
pattern used in `vagrant-stroller`. In a nutshell, a user should be able to compose entire infrastructure
stacks quickly and easily with Vagrant, all using potentially different options.

To accomplish this, Vagrant has been extended to read from a `stack` file. A `stack` file is simply
a YAML file defining what a given infrastructure stack looks like. You can add as many stacks
as you would like, and switch between them with minimal problems.

An example Vagrant Stack looks like:
```yaml
---
# Defaults can be defined and reused with YAML anchors
defaults: &defaults
  domain: stackstorm.net
  memory: 2048
  cpus: 1
  box: puppetlabs/ubuntu-14.04-64-puppet
# Define as many hosts as you would like. Deep merged in!
st2server:
  <<: *defaults  # Take advantage of YAML anchors and inherit
  hostname: st2server
  # Any number of facts available to the server can be set here
  puppet:
    facts: # Apply facts to the guest OS
      role: st2server
  mounts:
    # Mounts can be in a temp directory
    - /opt/stackstorm
    # Or with an absolute path
    - "/opt/st2web:/tmp/st2web"
  # Any number of private networks can be defined
  private_networks:
    - 10.20.30.2
```

To switch between a stack, simply change the Environment variable `stack` to something else. This
can be done at runtime...

```
stack=cicd vagrant status
```

... exported for a session ...

```
# All subsequent actions for the shell session will be this stack
export stack=cicd
vagrant status
```

... or set it up to stay configured indefinetely. This is done via the `dotenv` gem. Simply edit
the file `.env` in the top-level of this project, and add the line `stack=cicd` and you're done!

See more examples in the `stacks` directory at the top-level of this project.

### Supported Vagrant Objects

* Hostname [`hostname`]
* Domain [`domain`]
* Memory [`memory`]
* Private Networks [`private_networks`]
* Box name [`box`]
* Box URL [`box_url`]
* Puppet Facts [`puppet/facts`]
* Mounts [`mounts`]

### Digital Ocean Provisioning

In addition to having constructs to manage a `virtualbox` or `vmware_desktop` image, Vagrant Stacks
also supports using DO as a provisioner. To do this, you must set the following environment variables:

* `DO_SSH_KEY_PATH` - Path to SSH key used to log into servers
* `DO_TOKEN` - Digital Ocean Token used to provision servers

Then, in a Vagrant Stack, you can specify the following values:

```yaml
...
  do:
    image: '14.04 x64'
    region: nyc3
    size: 1gb
```

Refer to https://www.digitalocean.com/community/tutorials/how-to-use-digitalocean-as-your-provider-in-vagrant-on-an-ubuntu-12-10-vps
for more information on available options

## Provisioners

Support for many provisioners out-of-the box with the ability to install `st2`, and perform basic
tasks (install packages, configure users, setup CM downloads, etc) is a goal of this project. To
that end, you can change also at runtime the provisioner used to provision a server. Set the
environment variable `provisioner` at runtime, or in the `.env` file as outlined in the overview.

### `puppet-agent`

A script is copied to `/usr/bin/puprun`, which will be used to do branch updates based on upstream `git`,
and act as the masterless executor.

Puppet can run a series of environments, covered by `git` branching. To deploy a specific branch, simply
set the environment variable `environment` and off you go!

Example:
```
environment=myawesomechange puprun
```

The node will stay on the `myawesomechange` branch until:
* The branch is deleted from upstream, where it will automatically revert back to `production`, or...
* You specify another branch to run as illustrated above

#### Puppet Environments

Vagrant also able to be super flexible. By default, a branch known as `current_working_directory` is
created and used as the environment for Puppet to run in. This prevents you from having to `git commit`
and push upstream to make and test local changes.

Vagrant has the ability to mock out different nodes, as well as different environments. Simply use the
`hostname` and `environment` variables as appropriate.

Vagrant also has the ability to dynamically switch out Vagrant Baseboxes. Use the `box` environment
variable to change it up. (More details can be found inside the `Vagrantfile`)

Example:
```
hostname=ops001 box=fedora vagrant up
```

The node will remain named `ops001.stackstorm.net` and running on Fedora until it is destroyed
