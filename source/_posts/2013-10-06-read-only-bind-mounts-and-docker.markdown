---
layout: post
title: "Read only bind mounts and docker for unix domain sockets"
date: 2013-10-06 23:48
comments: true
categories: docker linux puppet
---

I've been playing around with [Docker](http://docker.io) a load recently, and I've developed a nice
little pattern for sharing sockets that I'm gonna pass on. (By sockets, I specifically
mean [Unix domain sockets](http://en.wikipedia.org/wiki/Unix_domain_socket)) for the purposes
of this post.

A lot of applications can either use a TCP socket, or a Unix domain socket to communicate - for
example [FastCGI](http://www.fastcgi.com/drupal/) and [http://www.mysql.com/](MySQL) both
allow you to use either mode (and in the case of mysql, both).

I have a small home server, and a want to run a bunch of applications that inside containers
(for security and management reasons), and these applications need to speak to a mysql server
(via a unix domain socket - which just appears to be a file on the filesystem.

I also want to run the mysql server inside a container - so the mechanics of getting a socket
shared between them are a little non-trivial.

Lets go through a worked example of how I've solved this for the dovecot imap and pop3 server
talking to mysql. This is an especially fun example, as dovecot runs as root (within it's container)
so if someone hacked into the server through an exploit in dovecot - they can rm -rf anything
I share with that container...

First off, I've created some [LVM](http://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29) volumes:

>  mysql        vg0  -wi-ao   4.00g

This is the mysql data directory, which will be mounted in the mysql container as /var/lib/mysql

>  mysql_socket vg0  -wi-ao   4.00m

This is going to get mounted at /socket/mysql inside containers, and will hold the mysql Unix domain socket

>  vmail        vg0  -wi-ao  20.00g

And this is my mail spool for dovecot with all the emails in it.

The immediate problem with this is that if the mysql socket volume is shared between multiple other containers
(as dovecot won't be the only app using this mysql instance), as dovecot runs as 'root', then if it gets hacked,
the hacker can delete the mysql socket, and any other programs trying to connect to mysql through that
socket will be affected.

We can't allow that - that would defeat the entire point of using containers for application isolation!

The trick is that you only need read access to use a unix domain socket that some other program has created.

Therefore, we can use [bind mounts](http://docs.1h.com/Bind_mounts) to fix this - by re-binding a readonly
(-o ro) copy of the file system, and giving *that* to dovecot would stop these issues, easy...

So, our partitions get mounted like this:

> /dev/mapper/vg0-vmail on /mnt/volumes/vmail type ext3 (rw,noatime)
> /dev/mapper/vg0-mysql on /mnt/volumes/mysql type ext3 (rw,noatime)
> /dev/mapper/vg0-mysql_socket on /mnt/volumes/mysql_socket type ext3 (rw,noatime)
> /mnt/volumes/mysql_socket on /mnt/volumes_ro/mysql_socket type none (ro,bind)

The last line involves a little trick. When I tried this, to my dismay, it didn't work.

The /etc/fstab entry associated with it is:

> /mnt/volumes/mysql_socket /mnt/volumes_ro/mysql_socket    none    bind,ro 00 00

But [as you can see](https://www.google.co.uk/search?q=bind+mount+read+only) from the Googles,
read only bind mounts are a bit tricky - you have to re-mount them a second time to make them
read-only.

The trick I use here is twofold - one, I only create them with puppet (mostly using the excelent
[mounts module](https://forge.puppetlabs.com/AlexCline/mounts) (which I only had to patch [a little bit](https://github.com/bobtfish/puppet-mounts/commit/abe73f82064f01bccd45289eb4f4ce51353ca364)),
the associated code looks like this:

{% codeblock lang:sh %}
  mounts { "Mount volume ${name} bind to ro":
    ensure => present,
    source => "/mnt/volumes/${name}",
    dest   => "/mnt/volumes_ro/${name}",
    type   => 'none',
    opts   => 'bind,ro', # Note bind mount ignores ro till you remount, thus the trickery below
  }
  ->
  exec { "/bin/mount -o remount,ro '/mnt/volumes_ro/${name}'":
    onlyif => "/bin/mount -l | /bin/grep '/mnt/volumes/${name} on /mnt/volumes_ro/${name} type none (rw,bind)'"
  }
{% endcodeblock %}

This could be notify rather than just ordering - but I like to be paranoiad and check these are ok
every puppet run..

However, if the machine is freshly rebooted, then the read-only file systems won't be remounted yet, ergo
the second piece of cunning is an upstart script to make sure things as kosher after a reboot:

{% codeblock lang:sh %}
# /etc/init/remount-ro-bind-mounts.conf - Dirty hack to remount ro bind mounts properly

description "Remount RO bind mount filesystems on boot"

start on local-filesystems

console output

script
  for fs in $(cat /etc/fstab | awk '{ print $2 " " $4 }' | grep 'bind,ro$' | cut -d' ' -f1); do mount | grep rw,bind | awk '{ print $3 }' | grep $fs >/dev/null; if [ $? -eq 0 ]; then mount -o remount,ro $fs; fi; done
end script
{% endcodeblock %}

And there we go - all done(-ish). I just rsync the vmail spool and mysql data from another machine, and start it all up!

The containers look like this:

{% codeblock lang:sh %}
$ sudo docker ps
ID                  IMAGE                   COMMAND             CREATED             STATUS              PORTS
04a9da050ba8        t0m/dovecot:latest      /start run          25 minutes ago      Up 25 minutes       110->110, 143->143, 993->993, 995->995
20b3a7a65966        t0m/mysql:latest        /start run          About an hour ago   Up About an hour
{% endcodeblock %}

And they get run with the upstart script from [Garth's excelent docker puppet module](https://forge.puppetlabs.com/garethr/docker):

{% codeblock lang:sh %}
$ ps aux | grep 'docker run' | grep -v grep
root     16874  0.0  0.0 273744  4944 ?        Ssl  Oct06   0:00 docker run -u mysql -v /mnt/volumes/mysql/data:/var/lib/mysql -v /mnt/volumes/mysql_socket:/socket -m 0 t0m/mysql:latest
root     28498  0.0  0.0 272336  4944 ?        Ssl  Oct06   0:00 docker run -v /mnt/volumes/vmail:/var/vmail -v /mnt/volumes_ro/mysql_socket/run:/socket/mysql -m 0 t0m/dovecot:latest
{% endcodeblock %}

And last but not least, here's it working:

{% codeblock lang:sh %}
$ openssl s_client -quiet -connect localhost:995
depth=0 C = GB, ST = Greater London, L = London, O = bobtfish.net, CN = mail.bobtfish.net
verify error:num=18:self signed certificate
verify return:1
depth=0 C = GB, ST = Greater London, L = London, O = bobtfish.net, CN = mail.bobtfish.net
verify return:1
+OK Dovecot ready.
USER bobtfish@bobtfish.net
+OK
PASS xxxxxx
+OK Logged in.
STAT
+OK 488 15003687
{% endcodeblock %}
