---
layout: post
title: "Terraform 0.4.0"
date: 2015-04-03 00:48
comments: true
categories: terraform, aws, devops
---

[Terraform 0.4.0 is out](https://hashicorp.com/blog/terraform-0-4.html), with the [remote modules](https://github.com/hashicorp/terraform/pull/1185) feature!
As per [my last post](http://localhost:4000/blog/2015/03/29/terraform-from-the-ground-up/),
I've been playing around with this already, and I thought it was worth showing
off my simple example of this feature actually being used.

It's wrapped up in a way that you can fork my 2 repositories and play with it
yourself, and I encourage [you to do so](https://github.com/bobtfish/terraform-example-vpc-infra/blob/master/eucentral1-demo/terraform.tfvars#L3)

<!-- more -->

If you take [the example VPC](https://github.com/bobtfish/terraform-example-vpc/tree/master/eucentral1-demo)
from my last post (with the state file committed), you can now pull in the data in another terraform
repository [like this](https://github.com/bobtfish/terraform-example-vpc-infra/blob/master/eucentral1-demo/vpc.tf)
and then use the outputs it defines [like this](https://github.com/bobtfish/terraform-example-vpc-infra/blob/master/eucentral1-demo/kubernates.tf#L8).

This is pretty neat, and it'll be super cool to allow multiple teams to work on different layers
of the infrastructure, using the [consul state store](https://www.terraform.io/docs/commands/remote.html) and [ACLs](https://www.consul.io/docs/internals/acl.html),
more news on that once I've had chance to play with it.

Unfortunately, the credentials file feature didn't make 0.4.0, and I [do math that isn't possible in master](https://github.com/bobtfish/terraform-aws-coreos-kubernates-cluster/blob/master/nodes.tf#L28) now, so you'll still need to use my fork if you want to use my examples verbatim.

I've written a bunch of modules to power my examples though, most of which work on unpatched terraform, and I thought that I'd
list them (in most to least reuseable order) so that you don't have to dig through all the code or my github to find them :)

  * [terraform-amitype](https://github.com/bobtfish/terraform-amitype) - Match an AMI type (e.g. m3.xlarge) to its virtualization type (e.g. hvm)
  * [terraform-azs](https://github.com/bobtfish/terraform-azs) - Work out which azs each of your accounts has access to for easy lookup/templating across accounts.
  * [tf_aws_ubuntu_ami](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami) - Look up Ubuntu [AMIs](http://cloud-images.ubuntu.com/locator/ec2/)
  * [tf_aws_coreos_ami](https://github.com/terraform-community-modules/tf_aws_ubuntu_ami) - Look up the most recent [CoreOS](https://coreos.com/) [AMI](https://coreos.com/docs/running-coreos/cloud-providers/ec2/)
  * [terraform-vpc](https://github.com/bobtfish/terraform-vpc) - Build out an initial VPC with 2 AZs
  * [terraform-vpc-nat](https://github.com/bobtfish/terraform-vpc-nat) - Build an initial NAT instance out on top of a VPC made with terraform-vpc
  * [terraform-aws-coreos-kubernates-cluster](https://github.com/bobtfish/terraform-aws-coreos-kubernates-cluster) - Kubernates cluster on CoreOS (n.b. not yet working out the box, needs patched terraform)

