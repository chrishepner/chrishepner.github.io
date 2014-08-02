---
layout: post
title:  "Developer-Specific Vagrant Configuration: Part II"
categories: [vagrant, ansible]
---

(This is a follow-up to a previous post about customizing Vagrant configurations. If you haven't read that, this won't
make much sense. Check that out [here]({% post_url 2014-07-30-vagrant-local-customization %}).)

## Adding database support

After adding the ability for developers to set their own folder syncing within a shared project, several people asked
about adding the same functionality for databases. We're in a situation where developers need separate credentials
for local development, or are working on one-off projects that would not make sense to provision for everyone.

I added a few lines to the local.yml.example file, as follows:

{% highlight yaml %}

# List of MySQL databases.
# This is provisioned after any accounts specified
# by the provisioning script.
databases:
  - name: testdb
    import: testdb.sql


# MySQL database users.
# Again, this is provisioned after any accounts
# specified after the provisioning script.
database_users:
  - name: testuser
    password: "testpass"
    privileges: "*.*:ALL" #optional, defaults to *.*:ALL


{% endhighlight %}

Once we're done modifying the Vagrantfile and Ansible playbook, this will provision a database, "testdb", import an SQL
dump `testdb.sql`, and ensure a user exists with name "testuser", password "testpass", and full privileges to all
databases.

This is just slightly more involved than setting up the folder forwarding. We have to pass this information to the
provisioner, Ansible, since this is what actually provisions the configuration of the machine. We'll edit
the Vagrantfile as follows:

{% highlight ruby %}
config.vm.provision "ansible" do |ansible|
    if local_exists then
        ansible.extra_vars = {
            database_users: local_config['database_users'],
            databases: local_config['databases'],
        }
    end
    ansible.playbook = "provisioning/site.yml"
end
{% endhighlight %}

This will pass the `database_users` and `databases` variables to Ansible using `ansible.extra_vars`, as documented
 [here](http://docs.vagrantup.com/v2/provisioning/ansible.html).

Now it's just a matter of using this information to provision the databases and their users:

{% highlight yaml %}
- name: Copy database dumps from local
  copy: src={{ item }} dest=/tmp/dbimport/
  with_fileglob:
  - "db/*"
- name: Setup MySQL databases from local.yml
  mysql_db: name={{ item.name }}
            state=import
            target="/tmp/dbimport/{{ item.import }}"
  when: item.import is defined
  with_items: databases
  sudo: no

- name: Setup MySQL users from local.yml
  mysql_user: name={{ item.name }}
              password={{ item.password }}
              priv={{ item.privileges|default('*.*:ALL', true) }}
              state=present
  with_items: database_users
{% endhighlight %}


The repo for this project is up on GitHub [here](https://github.com/chrishepner/my-lamp). If you're using a similar
solution or have found a better approach, let me know by email or in the comments below!