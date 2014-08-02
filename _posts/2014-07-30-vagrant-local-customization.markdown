---
layout: post
title:  "Developer-Specific Vagrant Configuration: Part I"
categories: [vagrant, ansible]
---

##Introduction to Vagrant

In my day job, I maintain a [Vagrant](https://www.vagrantup.com) box. Vagrant, if you're not familiar, makes
it incredibly easy to setup, configure, and provision virtual machines for development and testing purposes.

Vagrant, and [Ansible](http://www.ansible.com/), the tool I use for provisioning, are awesome. I work with a team of six
other developers and designers, most of whom have no experience with virtual machines or server configuration, and
Vagrant allows me to provide each of them a local, sandboxed environment for them to develop in that closely resembles the
production environment. When I release a new version of the box all they need to do is a
{% highlight bash %}
$ git pull
$ vagrant reload --provision
{% endhighlight %}

to update.

Vagrant makes it to [forward folders](https://docs.vagrantup.com/v2/synced-folders/basic_usage.html)
between the "host" and "guest" boxes 
(the host being your "native" operating system and the "guest" OS being the virtual machine being run by Vagrant).

Vagrant's configuration is defined in a file in the root of your project, named `Vagrantfile`. The setup of folder 
forwarding looks like this:

{% highlight ruby %}
Vagrant.configure("2") do |config|
  # Configuration goes here
  # ...
  
  config.vm.synced_folder "/Users/chrishepner/dev/mysite", "/var/www/mysite"
  config.vm.synced_folder "/Users/chrishepner/dev/work", "/var/www/work"
end
{% endhighlight %}

Here we have `~chrishepner/dev/mysite` on my local machine forwarding to `/var/www/mysite` on the Vagrant VM. This just 
means that any changes I made in `~/dev/mysite` are automatically synced with `/var/www/mysite` in the VM, and
vice-versa.

You can probably see the problem here: not every developer is going to have the code they want forwarded to the VM
on the same path. Since the `Vagrantfile` is part of the repository, it's problematic for developers to be
setting their synced folders in that file directly. Sure, we *could* force everyone to place
all their files in some `/vagrant` directory or something similar, but that idea makes me sad.

## Locally-Customized Folder Forwarding 

I initially tried to achieve this by setting up a single synced folder within the vagrant directory, like so:
{% highlight ruby %}
    config.vm.synced_folder "site/", "/var/www/"
{% endhighlight %}

I then intended to have everyone just add symlinks within this `site` folder to the directories they want synced.
This wasn't the best idea for several reasons, the most important being that Virtualbox, the hypervisor I was using,
*really* didn't want to cooperate with syncing symlinked directories, even when I set it's configuration to do so.
Time for a different approach.

Thankfully, the `Vagrantfile` is just Ruby, so we can add on to it as we please. My day job is primarily a PHP shop,
so I didn't want to require the team to write in a language they weren't familiar with. I created a `local.yml` file,
intended to be excluded from version control, where each person can add key-value pairs of host:guest directories they
want to sync. I modified the default `Vagrantfile` as follows:

{% highlight ruby %}
require 'yaml'


# ...

# Include local configurations for paths synced with the webroot
local_exists = false
if File.exists?(File.join(vagrant_dir,'local.yml')) then
    local_exists = true
    local_config = YAML.load_file(File.join(vagrant_dir, 'local.yml'))
end

# ...

config.vm.define "web" do |web|
    # ...
    if local_exists then
          local_config['synced'].each do | synced |
            web.vm.synced_folder synced['local_path'], synced['server_path'], :mount_options => [ "dmode=775", "fmode=774" ]
          end
    end
end

{% endhighlight %}

I also added a `local.yml.example` within the repo, which the developer just needs to copy to
`local.yml` and customize:

{% highlight yaml %}

---
#
# Configuration file for individual developer.
#


# "synced" defines folders to forward to the
#  virtual machine. local_path is the
#  path on your local machine, and server_path
#  is on the remove machine.
#
# After modifying this file, you will need to run 
# 'vagrant reload' for your changes to take effect.
synced:
  - local_path: /home/yourlocal/dev/ansible
      server_path: /var/www/html/first
  - local_path: /home/yourlocal/dev/otherpath
      server_path: /var/www/html/second


{% endhighlight %}

I've created a repository, "[my-lamp](https://github.com/chrishepner/my-lamp)" using this pattern. (This
example provisions a simple LAMP stack, but the logic detailed above does not depend on PHP or Apache.) In [the next
article]({% post_url 2014-08-02-vagrant-local-customization-2 %}), I'll discuss adding support for adding MySQL databases and their user accounts.