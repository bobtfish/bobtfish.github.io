---
layout: post
title: "Terraform from the ground up"
date: 2015-03-29 21:18
comments: true
categories: terraform, aws, devops
---

I've been playing around with [Terraform](https://www.terraform.io/) a bunch recently, and I'm pretty [excited](https://twitter.com/bobtfish/status/581905067948797952) about [0.4.0](https://github.com/hashicorp/terraform/blob/master/CHANGELOG.md#040-unreleased).

However at the moment, the examples leave quite a lot to be desired. So for my own learning and entertainment, I've created a set of example Terraform modules,
and a simple example that you should be able to clone and run.

<!-- more -->

## What's exciting in Terraform

One of the [pull requests](https://github.com/hashicorp/terraform/pull/1319) [I've been watching](https://github.com/hashicorp/terraform/pull/1080) [https://github.com/hashicorp/terraform/pull/1076](around ASGs) / etc were merged recently, [fixing Tags on ASGs](https://github.com/hashicorp/terraform/issues/932) which I use at work for the [puppet ENC](https://docs.puppetlabs.com/guides/external_nodes.html), and the [support for ~/.aws/credentials](https://github.com/hashicorp/terraform/pull/1049) files is almost ready.

In fact, after chatting with folks at [scalesummit](http://www.scalesummit.org/) on Friday, I was so excited that I wanted to pull some of the new features into
[my fork](https://github.com/bobtfish/terraform) and play with them.

One of the most exciting strategic features to me is what's been called '[remote modules](https://github.com/hashicorp/terraform/pull/1185)',
which when it's merged will allow you to consume external Terraform state (in a read-only way) from other Terraform repositories.

This is *almost exactly* one of the features I'd sketched out as something Terraform needed to allow it to scale as a tool for larger organisations.

At my day job, we have multiple teams managing different parts of the infrastructure - for example one team is responsible for VPCs
and DNS/puppet masters etc, whilst another team is responsible for kafka/zookeeper clusters, and several other teams all having
independently managed Elasticsearch clusters.

Ergo the ability to 'publish' state between teams and thus allowing different teams to share the basics like VPCs and subnets)
whilst having their own configs is essential for the variety of workflows and machine lifecycles that I need to support.

I haven't yet tested out this functionality, as I [got distracted](http://www.globalnerdy.com/wordpress/wp-content/uploads/2012/09/yak-shaving.jpg)
making some example 'base infrastructure' modules that I wanted to consume data from.

## Unreleased software note!

You need to use [my fork](https://github.com/bobtfish/terraform) of Terraform for the examples below to work,
but I expect them to work without any significant changes in 0.4.0

## Terraform modules

I've scratched my head about how to do Terraform modules (which need to lookup external data) for a while, and I've come up with a pattern that I don't hate.

I've also written some public modules that are on github, which you can look at and criticise. (Sorry in advance for the terrible ruby)

Whilst I haven't yet used the feature I merged, the state from the example in this post is a semi-realistic example VPC
that I'll be able to use for further testing.

To give you some insight into how I've put things together, I'm going to dive into
each of the modules and explain a little about them, before
showing how I wrap them up together with the actual code/config to launch a VPC.

### Looking up AMIs

Lets think about how we should lookup a Ubuntu AMI using [a module](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami).

There's [a giant table](http://cloud-images.ubuntu.com/locator/ec2) of AMIs available on the web, and the data for it is [almost](http://cloud-images.ubuntu.com/locator/ec2/releasesTable), [but not quite](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami/blob/master/getvariables.rb#L11) JSON you can parse.

So I wrote a getvariables.rb script, a [Makefile](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami/blob/master/Makefile) which generates [variables.tf.json](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami/blob/master/variables.tf.json) - end result, we have a giant hash table of all the AMIs for Ubuntu in all of the regions.

We can then use a combination of variables and the [lookup + format functions](https://github.com/bobtfish/terraform/blob/master/website/source/docs/configuration/interpolation.html.md#user-content-built-in-functions) in [the main.tf](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami/blob/master/main.tf) to output the desired AMI.

Then we just use the module, supplying it the params it needs, and we get the right AMI.

```
module "ami" {
  source = "github.com/terraform-community-modules/tf_aws_ubuntu_ami"
  region = "eu-central-1"
  distribution = "trusty"
  architecture = "amd64"
  virttype = "hvm"
  storagetype = "instance-store"
}

resource "aws_instance" "web" {
  ami = "${module.ami.ami_id}"
  instance_type = "m3.8xlarge"
  ...
}
```

### What availability zones?

Next example - in AWS, which availability zones you're given access to depends on your account.

Therefore, to provide a generic template that anyone can use to launch a VPC (in any region), we need to be able
to detect which regions and availability zones a user has access to.

Using almost the same pattern, and [the aws cli tool](http://aws.amazon.com/cli/), I've produced another module
which will read your [~/.aws/credentials file](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files) to
pull in a list of all your available availability zones.

Due to the way Terraform currently handles things, [this module](https://github.com/bobtfish/terraform-azs) just exports three variables, a primary, secondary, and (where available)
tertiary availability zone (ordered by alphabetical sort).

```
module "az" {
  source = "github.com/terraform-community-modules/tf_aws_availability_zones"
  region = "eu-central-1"
  account = "demo"
}

resource "aws_subnet" "primary-front" {
  availability_zone = "${module.az.primary}"
}
```

Note that for this to work, you will need a _[demo]_ section in your ~/.aws/credentials file, as the variables.tf.json file
is *not* committed to the repository, unlike the last example.

### Lets build some damn infrastructure already!

We've now got the pieces we need to put together a VPC, so lets go ahead and [write a module to do that](https://github.com/bobtfish/terraform-vpc).

This takes a /16 network and builds out a VPC with public (internet IP), private (fixed address) and ephemeral (ASG/ELB) subnets
in two availability zones.

Note that we have *a lot* [of outputs here](https://github.com/bobtfish/terraform-vpc/blob/master/outputs.tf), as we need to expose
every piece of infastructure we want to be able to reference to the module's users.

### But I meant an actual machine!

That's the next step, we want to launch [our first NAT machine](https://github.com/bobtfish/terraform-vpc-nat/blob/master/main.tf#L47), and again, we'll [write a module for it](https://github.com/bobtfish/terraform-vpc-nat)!

We use [cloud-config](https://help.ubuntu.com/community/CloudInit#Cloud_Config_Syntax) to setup a firewall on the initial machine, so that subsequent
machines (without public IP addresses) can NAT externally, and we reset the main routing table (used by the 'back' and 'ephemeral' subnets created
by the VPC module) to be one which points to the new NAT instance. For [good measure](https://pbs.twimg.com/media/CBHd3OVUwAA8FBh.png:large) we also install [Docker](https://www.docker.com/) and [puppet](https://puppetlabs.com/).

Note that we have to use the remote_exec provisioner here to wait for cloud-init to finish, so that we know the firewall rules are in place
for NAT before we launch any more machines

## So when do we get to do something I can actually run?

Ok, lets put the VPC + nat machine module to work, and [build a host inside the private
subnets](https://github.com/bobtfish/terraform-example-vpc/blob/master/eucentral1-demo/internal.tf).

You can (and should!) fork and clone [this example](https://github.com/bobtfish/terraform-example-vpc), which contains
just enough stuff to string the modules we've written together and bring up a couple of hosts.

You'll need:

  * [my fork](https://github.com/bobtfish/terraform) of Terraform
  * [An ~/.aws/credentials file](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files) with a _[demo]_ section

Lets go:

```
~ ⭐ git clone git@github.com:bobtfish/terraform-example-vpc.git
Cloning into 'terraform-example-vpc'...
remote: Counting objects: 61, done.
remote: Compressing objects: 100% (34/34), done.
remote: Total 61 (delta 27), reused 54 (delta 20), pack-reused 0
Receiving objects: 100% (61/61), 15.01 KiB | 0 bytes/s, done.
Resolving deltas: 100% (27/27), done.
Checking connectivity... done.
~ ⭐ cd terraform-example-vpc/
terraform-example-vpc (master👆 ✔) ⭐ make
ssh-keygen -t rsa -f id_rsa -N ''
Generating public/private rsa key pair.
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
The key fingerprint is:3d:c8:0c:76:1e:29:25:a2:52:cd:7d:09:9b:53:24:c9 t0m@somebox
The key's randomart image is:
+--[ RSA 2048]----+
|  .+.o+o+.       |
| . .E..Bo.       |
|. .   B.+        |
| .   . O +       |
|        S o      |
|           o     |
|                 |
|                 |
|                 |
+-----------------+
true
terraform-example-vpc (master👆 ✔) ⭐
```

This has setup an ssh key to allow you to log into the created instances.

Now, change directory into the region+account folder ([Terraform can only do one region at once currently](https://github.com/hashicorp/terraform/pull/1281)),
and run make again. This pulls in all the necessary modules (using terraform get), and then runs their Makefiles by iterating over .terraform/modules to ensure that _variables.tf.json_ is built it needed.

Note that even modules which don't need to build _variables.tf.json_ are required to have a Makefile (which can do nothing).

```
eucentral1-demo (master👆 ✔) ⭐ make
terraform get
Get: git::https://github.com/terraform-community-modules/tf_aws_ubuntu_ami.git
Get: git::https://github.com/bobtfish/terraform-vpc-nat.git
Get: git::https://github.com/bobtfish/terraform-vpc.git
Get: git::https://github.com/terraform-community-modules/tf_aws_ubuntu_ami.git
Get: git::https://github.com/bobtfish/terraform-azs.git
for i in $(ls .terraform/modules/); do make -C ".terraform/modules/$i"; done
make[1]: Entering directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/1b2917ba643e4bf90b900b6da59b2e05'
ruby getvariables.rb
make[1]: Leaving directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/1b2917ba643e4bf90b900b6da59b2e05'
make[1]: Entering directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/3f6892e390615d3c9f3cc69d65d7291c'
true
make[1]: Leaving directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/3f6892e390615d3c9f3cc69d65d7291c'
make[1]: Entering directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/be339b8825e57c8cb0372c530eaf49bf'
true
make[1]: Leaving directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/be339b8825e57c8cb0372c530eaf49bf'
make[1]: Entering directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/d4386a9079b869044647990c72b8f0e4'
make[1]: Nothing to be done for `all'.
make[1]: Leaving directory `/home/t0m/terraform-example-vpc/eucentral1-demo/.terraform/modules/d4386a9079b869044647990c72b8f0e4'
ruby getvariables.rb > variables.tf.json
eucentral1-demo (master👆 ✔) ⭐ terraform apply
aws_key_pair.deployer: Creating...

... loads more stuff deleted ...

Outputs:

  nat-public-ip = 52.28.22.233
```

You should be able to ssh to your nat instance:

    ssh-add ../id_rsa
    make sshnat

and from there, ssh into your back network host:

    ssh 10.1.1.4

and you should see [consul](https://www.consul.io/) running (in Docker [of course](https://pbs.twimg.com/media/CBHd3OVUwAA8FBh.png:large))

```
ubuntu@ip-10-1-1-4:~$ ping -c 2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=56 time=6.74 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=56 time=6.96 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 6.746/6.856/6.966/0.110 ms
ubuntu@ip-10-1-1-4:~$ sudo docker ps
CONTAINER ID        IMAGE                  COMMAND                CREATED             STATUS              PORTS                                                      NAMES
d63b2034bf80        fhalim/consul:latest   "/bin/sh -c '/usr/lo   6 seconds ago       Up 6 seconds        8400/tcp, 0.0.0.0:8500->8500/tcp, 0.0.0.0:8600->8600/udp   consul
```

## Reusing

As most of the logic is in modules, to create another VPC in another environment or account, the top level directory can just be copied, 
the terraform.tfstate file removed and the details in _terraform.tfvars_ updated.

## Conclusion

Whilst a bunch of stuff in the current demo is more simplistic than a real infrastructure, it shows that Terraform can be used to
bootstrap entire VPCs with non-trivial configurations.

I haven't (yet) installed anything 'real' other than a docker container and some iptables rules as cloud-init is a terrible
config management mechanism! However the current example should be useful as a platform for jumping off from, either
on top of pre-existing infrastructre and puppet masters/chef servers (add vpc peering or vpns), or with ssh based config management
(ansible, salt).

## What's next?

In the next post (or rather, in my next set of hacking that I may or may not write up ;), I plan to use the infrastructure details of this subnet to test out
the remote module feaure, try to get a puppetmaster (with EC2 tags as the ENC) up, and explore storing the state in [consul](https://www.consul.io/).

