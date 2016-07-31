---
title: Managing your Digital Ocean Droplets using Ansible
date: 2016-07-31 17:30:50
tags:
---

## The Problem

There are a series of steps required before you can deploy an applicatoin to the cloud. You must first pick an operating system and start up a server using your cloud provider's interface. Then you must ssh into the machine and install all the typical libraries, dependencies, and packages that your application server needs. This process seems to work great for small pet projects, but imagine what happens if you need to restart your server from scratch? You would have to re-do everything manually which takes time and of course effort. A shell script might be helpful in automating some common tasks:


``` sh
apt-get update
apt-get install nginx
apt-get install pip
pip-install django
apt-get install postgresql
```

But you would still need to figure out how to copy over application configuration files or check the status of background services. You would still need to manually login to your cloud provder's user interface to spin up a virtual machine. You would still need to ssh into the machine and manually copy over your script file to run. This is manual/tedious work and it becomes much more difficult when you have to manage a large number of servers on your own.

## Ansible - automate everything

[Ansible](https://www.ansible.com/) is a configuration management software that helps you automate your IT infrastructure. It basically works by reading a yml file of instructions that describes exactly what actions you want to perform on a remote machine. It uses SSH to automatically connect to a machine and then performs the actions defined in your yml file. Since it connects to the cloud machine on its own, you should never access the remote terminal when initially setting up your server. 

There are a lot of concepts about Ansible that this blog post is not going to cover. The best way to see the value of Ansible is with a live example.



## Requirements

###### Sample Project

A [sample project](https://github.com/harneksidhu/blog-examples/tree/master/ansible-digital-ocean) with an Ansible infrastructure is provided for your convenience. This should be cloned onto your machine. The project utilizes a virtual machine to encapsulate Ansible as well as any required modules that it needs in order to operate. The virtual machine acts as the control machine from which all ansible commands will be performed. We could have avoided the virtual machine all together and just installed Ansible on your host operating system, but I found that having a consistent environment is beneficial especially if you want to be able to work on a variety of different operating systems.

###### VirtualBox/Vagrant

You will need to do is install [Virtualbox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.virtualbox.org/wiki/Downloads). Virtualbox provides you with the ability to run a virtual machine on your host operating system. You can think of Vagrant as a utility software that wraps around VirtualBox and provides you with the ability to provision a virtual machine in an automated way. Click [here](https://www.vagrantup.com/docs/provisioning/) if you would like to learn more about how this works and the benefits of provisioning development environments.

###### DigitalOcean

Create an account DigitalOcean if you have not already. If you would like to have a $10 credit, you can use my referall link [here](https://m.do.co/c/385c5ec4be11). After creating your account, you will need to generate an [API token](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2) that Ansible can use in order to send requests to your account.

###### Create public/private ssh keys

The sample project requires a public/private ssh key pair. You could use `ssh-keygen` or any alternative software you feel comfortable with.

## Organize Project

With the sample project cloned onto your machine, you will need to perform the following steps in order for the demo to work:

1. Edit the `group_vars/all.yml` file and replace `<Enter_API_Token>` with your Digital Ocean api token
2. Place the ssh public/private key pair under the `keys/` folder

Your project directory should now look like this:

{% asset_img project-structure.png %}

After that, all you have to do is run 

``` sh
vagrant up
```

on your favourite terminal client to spin up the virtual machine. This should take a while as it downloads an ubuntu image, provisions it with ansible, and installs all the required modules that ansible needs in order to operate the demo.


## Run the demo











3. Open a terminal and change the directory to where this repository is located.
4. Run `vagrant up` and wait till the machine has been provisioned.
5. Run `vagrant ssh` to acces the virtual machine. You will be logging in as user `vagrant` and the password is `vagrant`.
6. Once inside the machine, run `ansible-playbook setup.yml` to run the demo and `ansible-playbook destroy.yml` to destroy it.



