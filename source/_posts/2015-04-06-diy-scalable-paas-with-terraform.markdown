---
layout: post
title: "DIY Scalable PAAS with Terraform"
date: 2015-04-06 23:45
comments: true
categories: terraform, aws, devops
published: false
---

I've been ranting about [Terraform](https://www.terraform.io/) [a](/blog/2015/03/29/terraform-from-the-ground-up/) [lot](/blog/2015/04/03/terraform-0-dot-4-0/) recently, and I've shown some pretty
neat examples of building a VPC - however before now I haven't got anything *actually*
useful running.

In this post, we're gonna change that, and launch ourselves a publicly
available and scalable PAAS (Platform As A Service), built around Apache [Mesos](http://mesos.apache.org/) and [Marathon](https://github.com/mesosphere/marathon)
with [nginx](http://nginx.org/) and Amazon's [ELB](http://aws.amazon.com/elasticloadbalancing/) for load balancing and [Route53](http://aws.amazon.com/route53/) for DNS.

<!-- more -->

## Getting setup

You'll need an AWS account, a Ubuntu machine and a domain you control (which doesn't have to be in AWS) for this demo.
My example domain is _notanisp.net_, and so the subdomain I'm going to use in all the examples is _mesos.notanisp.net_

You'll also need the [AWS cli](http://aws.amazon.com/cli/) and an [~/.aws/credentials](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files)
file with a _[demo]_ section in it.

The easiest way to get my patched version of terraform is to build it in vagrant:

```
    git clone https://github.com/bobtfish/terraform.git
    cd terraform
    vagrant up
    vagrant ssh
    sudo apt-get install -y ruby ruby-json python-pip
    sudo pip install awscli
    export PATH=$PATH:/opt/go/bin:/opt/gopath/bin
    mkdir -p /opt/gopath/src/github.com/hashicorp
    cp -r /vagrant /opt/gopath/src/github.com/hashicorp/terraform
    cd /opt/gopath/src/github.com/hashicorp/terraform
    make updatedeps
    make dev
    cd ~/
    git clone https://github.com/bobtfish/terraform-example-mesos-cluster
    mkdir ~/.aws
    vi ~/.aws/credentials
```

Adding something like this:

    [demo]
    aws_access_key_id=AAAAAAAA
    aws_secret_access_key=BBBBBBBBBBBBBBBBB

Or if you prefer (and you trust me!), [here's a .deb[(http://notanisp-packages.s3-website-eu-west-1.amazonaws.com/dists/precise/main/binary-amd64/terraform_0.4.0-bobtfish0_amd64.deb)
you should be able to just install:

## Building your cluster

Once you're all setup, you can get started building your cluster:

```
cd terraform-example-mesos-cluster
make
```

This generates an ssh key (stored in _id_rsa_) to be able to ssh into your instances,
and captures your IP (stored in _admin_iprange.txt_) to allow access to the Mesos and
Marathon admin interfaces later. Note: You can edit the admin_iprange.txt file to
allow more IPs access, but it is *not* recommended to change this to 0.0.0.0/0,
as otherwise random people on the internet will be able to run jobs on your Mesos
cluster!

Lets go ahead and build out the cluster:

```
cd eu-central-1
vi terraform.tfvars # Adjust the 'domain' value to a subdomain
make # This pulls down all the terraform modules you need, and builds a map of which AZs your account can see.
terraform apply
```

The last step will take a while, go grab a coffee.

Eventually, terraform will finish, with output like this:

```
Apply complete! Resources: 35 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

  marathon_api  = http://marathon.admin.mesos.notanisp.net
  nat_public_ip = 52.28.37.228
```

Once it's finished, go into the route 53 control panel, and you should see your subdomain zone.
Go and grab its NS records:

![Route53 console](https://raw.githubusercontent.com/bobtfish/terraform-example-mesos-cluster/master/route53.png)

and put those into whatever you use to manage your main domain as a delegated record.

## Gettins admin pages and launching an app

After DNS catches up, you should be able to resolve:

  * mesos.admin.mesos.notanisp.net
  * marathon.admin.mesos.notanisp.net
  * www.mesos.notanisp.net

Hit http://marathon.admin.mesos.notanisp.net with your browser, and you should be able to see the
Marathon admin screen. (And you can see the Mesos admin screen at http://mesos.admin.mesos.notanisp.net)

You can now [launch an app](https://mesosphere.com/docs/tutorials/run-services-with-marathon)!

I've automated the launching of an example application, so you can just say:

    make deploywww

This will deploy [the hello world app](https://github.com/bobtfish/terraform-example-mesos-cluster/blob/master/eucentral1-demo/marathon_www.json)
 (from the mesosphere tutorial linked above) named '/www'
into marathon, and after a couple of mins, you should be able to access it from www.mesos.notanisp.net

Any additional apps you launch in Marathon with names like /someapp will be automatically bound
to a vhost on the load balancer. Pretty neat, eh?

## What we're building behind the scenes.

We [ask for a VPC to be built](https://github.com/bobtfish/terraform-example-mesos-cluster/blob/master/eucentral1-demo/vpc.tf)
(with the [module I created](https://github.com/bobtfish/terraform-vpc-nat) in [a previous post](http://bobtfish.github.io/blog/2015/03/29/terraform-from-the-ground-up/)).

This gives us 'front' and 'back' networks, and a [NAT instance](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_NAT_Instance.html).

On top of that, we build out a [mesos cluster](https://github.com/bobtfish/terraform-example-mesos-cluster/blob/master/eucentral1-demo/mesos.tf)

This comprises a number of pieces, first of all [masters](https://github.com/bobtfish/tf_aws_mesos/blob/master/mesos_master/master.conf), which
run [zookeeper](https://zookeeper.apache.org/) in addition to Marathon and Mesos. We run 3 of these by default, so one can fail without
affecting the operation of the cluster. Also there are [slaves](https://github.com/bobtfish/tf_aws_mesos/blob/master/mesos_slave/slave.conf)
which actually run the Mesos workers and excute applications. We launch 5 of these, and if one fails then any tasks it was running
will be re-launched on the remaining slaves (assuming they have enough resources to do so!).

Then we deploy 2 [load balancers](https://github.com/bobtfish/tf_aws_mesos/blob/master/lb/lb.conf) which discover the running applications from the marathon
API and configure nginx, with a publicly accessible [ELB](http://aws.amazon.com/elasticloadbalancing/) [in front of them](https://github.com/bobtfish/tf_aws_mesos/blob/master/elb/main.tf)
 to provide HA, in the event one of the load balancer machines fails.

We push this [into DNS](https://github.com/bobtfish/tf_aws_mesos/blob/master/dns/main.tf), along with records for the _adminlb_ machine.
[This machine](https://github.com/bobtfish/tf_aws_mesos/blob/master/elb/main.tf) is what proxies the mesos and marathon admin pages (and access to it
is locked down to your _admin_iprange.txt_)

## Redundancy and scaling

The cluster should be redundant to a load balancer or a mesos master failing. If the instance which fails can't be brought back
online, it should be possible to just terminate the instance and have terraform re-create it.

Also, you can eaisly adjust the number of slaves dynamically (just by [changing the variable](https://github.com/bobtfish/terraform-example-mesos-cluster/blob/master/eucentral1-demo/mesos.tf#L6)
 and running terraform), or the instance types employed (by rebuilding the cluster) if you'd like to run more tasks, serve real
load from the cluster, or scale whilst already serving real load!

Marathon apps which have already been deployed can be scaled up (to more instances) just by using the Marathon API with a web browser - I encourage you to try this, scaling
the example app up to 5 instances, or down to 1 instance and then killing 4/5 of the slaves.

## TODOs

Whilst the cluster we build is redundant, currently all the machines are allocated in a single availability zone.
I'll be investigating if I can get a way of generically deploying the cluster across two (or three if available) availability zones.

Also, currently the machines have no configuration management (no puppet/chef), which means that making any changes to them (or getting
any security updates) involves rebuilding the instances.

## Conclusion

Hopefully this post has shown that Terraform is already grown up enough to be used to compose real, production ready infrastructures.

The example shown here *is not* something I'd run as a production system, and there are a number of significant warts
in the code (like having to pack and unpack lists into comma seperated strings), but there's enough of the essential complexity
represented here to be close to 'real' applications and deployment, and I'm still having a great time exploring Terraform's
rapidly growing capabilities, and producing reuseable modules.

Whilst I don't think that anything but my most basic level of modules (e.g. AMI lookups) are likely to be reused verbatim,
I hope this example shows that it's possible to use terraform as a turnkey component in building repeatable
and composable infrastructures.

