---
layout: post
title:  "Terraforming Minecraft - Managing Configuration with Ansible"
date:   2020-11-04 0:00:00 -0700
categories: devops ansible linux minecraft
series: minecraft-terraform
order: 4
---

So we have the makings of a pipeline to take a minimal CentOS image, install software and create our own custom AMI's to launch any number of functional Minecraft servers in AWS EC2 instances. As it stands every server would be identical, but not all servers and instances are created equal. Servers with more users will require a larger instance and will need to be configured to take advantage of the extra hardware. Ansible is the tool we'll use for managing configuration on active instances.

> Ansible is often called a **provisioning** tool, but I think we should make a distinction between **provisioning**, i.e. creating infrastructure like AWS VPCs and instances, and **configuration management** which manages the configuration of the operating system on the instance. While Ansible is fully capable of launching instances and other AWS resources it is not ideal. Like with Terraform, the strength of Ansible is in it's declarative, idempotent style which allows us to describe our desired configuration without making any assumptions about it's current state. Ansible's declarative style works well for configuration but starts to break down when trying to provision infrastructure, which is where Terraform really shines.

### Creating an Ansible Project

Lets start by creating a very basic ansible project starting with the inventory file. Create a directory `ansible` in your project directory and create a new file `inventory.yaml`. In the inventory we'll define a `minecraft` group that will include our current instance and any others we create in the future. We'll also define a couple of variables for the `minecraft` group to tell Ansible how to access the instances.

{% highlight ini %}
[minecraft]
minecraft.pdizz.com

[minecraft:vars]
ansible_ssh_user=centos 
ansible_ssh_private_key_file=~/minecraft.pem
{% endhighlight %}

We also need a playbook to configure the instance. In the `ansible` directory create another file called `playbook.yaml` that will run, as root, on all hosts in the `minecraft` group.

{% highlight yaml %}
---
- hosts: minecraft
  become: yes
  become_user: root

  roles:
    - minecraft
{% endhighlight %}

Now we need to create the `minecraft` role with the file `ansible/roles/minecraft/tasks/main.yaml`

{% highlight yaml %}
---
- name: eula is accepted
  copy:
    content: "eula=true"
    dest: /opt/minecraft/eula.txt
    owner: minecraft
    group: minecraft
    mode: 0644

- name: minecraft service file is templated
  template:
    src: minecraft.service.j2
    dest: /usr/lib/systemd/system/minecraft.service
    owner: root
    group: root
    mode: 0644
  notify: minecraft is restarted # handler to restart service when config changes

- name: minecraft is running and enabled
  service:
    name: minecraft
    state: started
    enabled: true
{% endhighlight %}

This playbook declares that the eula file will exist with the contents "eula=true" (a prerequisite for running the server), a template will be used for the Minecraft service config, and the Minecraft service will be running, and also enabled so it will run whenever the instance restarts.

Like with most services, if we make any changes to the Minecraft service config we will need to restart the service for those changes to take effect. To do this we can to define a *handler* that Ansible will run when it detects a change in the configuration. Notice in the Ansible task `minecraft service file is templated` it is configured to `notify: minecraft is restarted` handler if the configuration changes. In a new file `ansible/roles/minecraft/handlers/main.yaml` we can define this handler

{% highlight yaml %}
---
- name: minecraft is restarted
  service:
    name: minecraft
    state: restarted
{% endhighlight %}

Now if and only if the task `minecraft service file is templated` detects a change to the file, Ansible will run the handler and restart the Minecraft service. Speaking of templates, we still need to create the template for the service config in `ansible/roles/minecraft/templates/minecraft.service.j2`

{% highlight ini %}
# /usr/lib/systemd/system/minecraftd.service
[Unit]
Description=The Minecraft Server
After=network.target

[Service]
Type=simple
User=minecraft
WorkingDirectory=/opt/minecraft
ExecStart=/usr/bin/java -Xms3072M -Xmx3072M -XX:+UseG1GC -jar server.jar --nogui
Restart=on-failure

[Install]
WantedBy=multi-user.target
{% endhighlight %}

So our ansible project directory should look like this

{% highlight text %}
ansible/
|-- roles
|   `-- minecraft
|       |-- handlers
|       |   `-- main.yaml
|       |-- tasks
|       |   `-- main.yaml
|       `-- templates
|           `-- minecraft.service.j2
|-- inventory.yaml
`-- playbook.yaml

{% endhighlight %}

Now we can run our Ansible playbook and configure the instance. In the ansible directory run `ansible-playbook playbook.yaml -i inventory.yaml -l minecraft.pdizz.com`

### Configuring a Server for Different Instance Sizes

We started off running our server on a `t3a.medium` instance with 2x2.5GHz CPUs and 4GB of RAM. According to these [Dedicated Server Requirements](https://minecraft.gamepedia.com/Server/Requirements/Dedicated) for a linux server that falls in the "Acceptable" range if we allocate 3GB of RAM for the server. We could spend a little more and upgrade our instance to something like `m5a.large` with 8GB of RAM. This would put us in the "Good" range for a dedicated server.

> To change your EC2 instance size on the fly, just update the instance type in your `ec2.tf` Terraform definition and run `terraform apply`. Terraform will stop the instance, change the instance type to whatever you define, and restart the instance to apply the change

To take advantage of the added hardware we'll need to update the template for our Minecraft service config. Eventually we might be running multiple servers on different sized instances so we should add a couple variables to our Minecraft service template using Ansible/Jinja template variable syntax {% raw %}`{{ my_var }}`{% endraw %}

{% highlight ini %}
# /usr/lib/systemd/system/minecraftd.service
[Unit]
Description=The Minecraft Server
After=network.target

[Service]
Type=simple
User=minecraft
WorkingDirectory=/opt/minecraft
ExecStart=/usr/bin/java -Xms{%raw%}{{ java_initial_heap_size }}{%endraw%} -Xmx{%raw%}{{ java_max_heap_size }}{%endraw%} -XX:+UseG1GC -jar server.jar --nogui
Restart=on-failure

[Install]
WantedBy=multi-user.target
{% endhighlight %}

We can define values for these variables for each host in our Ansible `inventory.yaml` with different values for each host

{% highlight text %}
[minecraft]
minecraft.pdizz.com      java_initial_heap_size=6144M java_max_heap_size=6144M
okay-minecraft.pdizz.com java_initial_heap_size=3072M java_max_heap_size=3072M
tiny-minecraft.pdizz.com java_initial_heap_size=1024M java_max_heap_size=1024M

[minecraft:vars]
ansible_ssh_user=centos 
ansible_ssh_private_key_file=~/minecraft.pem
{% endhighlight %}

Now when we run our playbook again the result of `minecraft service file is templated` should be `changed` and the `minecraft is restarted` handler should run to apply those changes.

> There are countless ways to organize an Ansible project, from putting everything in one file, to splitting group and host vars into separate directories. This is just one example of a basic project setup that should be adequate for a project this size

### Declarative Style and Idempotency in Ansible Playbooks

You may have noticed the funky naming convention I'm using for Ansible tasks like `eula is accepted` and `minecraft service file is templated`. Tasks names are arbitrary (you dont even need to name tasks at all) but I think it helps to train our brains to read and think in a *declarative* style. We should declaratively state "the file exists" instead of imperatively "create the file", or "the service is running" instead of "start the service", and let Ansible figure out the rest. 

The most reliable Ansible playbooks are totally idempotent. We should be able to run them any number of times, without assuming anything about the current state of the configuration, and always end up with the desired result. Playbooks that depend on the current configuration being in a certain state are very error prone. If your playbooks are littered with conditionals, or if you are afraid to run a playbook because it might do something unexpected, you're doing it wrong.

Run your playbooks early and often. Inspect the log for any unexpected changes. When you have confidence in your playbooks, configuration management can be an empowering experience instead of a nightmare of complexity and doubt.
