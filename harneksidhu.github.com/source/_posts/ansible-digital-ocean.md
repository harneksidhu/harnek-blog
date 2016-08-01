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

The sample project is a simple ansible script that performs the following actions (summarized):

1. Instructs DigitalOcean to create a droplet
2. Installs Nginx HTTP server
3. Configures Nginx to serve a static web page

To see this in action, run the following commands:


``` sh
vagrant ssh
cd ansible-digital-ocean/
ansible-playbook setup.yml
```

All we did was we connected to the local virtual machine which has ansible installed using ssh. We then changed the directory on the machine to where our ansible script (setup.yml) is stored. We finally used the `ansible-playbook setup.yml` command to fire the script. 

You should see a similar output below if all goes well:


{% asset_img playbook-output.png %}


If you head on over to your DigitalOcean dashboard, you should see a droplet that has been automatically created for you:

{% asset_img digital-ocean-dashboard.png %}


Since the server is running, you should be able to hit the ip-address on your browser to see Nginx serving a basic HTML page.



## Explanation

Let's browse through `setup.yml` and explain what is happening step by step. `setup.yml` is defined as a playbook. A playbook is composed of one or more `plays`. The goal of a `play` is to execute a set of tasks to a group of hosts. `setup.yml` is composed of three plays. The first play communicates with DigitalOcean to create a droplet. The second play connects to the droplet to install `python2` which is required for Ansible to operate correctly. The third play installs and configures Nginx.


The following snippet contains the first play:

``` yml
---
- hosts: localhost
  tasks:
  - name: Store public ssh key
    shell: cat keys/id_rsa.pub
    register: public_key
  - name: Store public ssh key in digital ocean
    digital_ocean:
      state: present
      command: ssh
      name: ansible_ssh_key
      ssh_pub_key: "{{ public_key.stdout }}"
      api_token: "{{ digital_ocean_api_token }}"
      unique_name: yes
    register: my_key
  - name: Create droplet
    digital_ocean:
        state: present
        command: droplet
        name: host0
        api_token: "{{ digital_ocean_api_token }}"
        size_id: 512mb
        region_id: tor1
        image_id: ubuntu-16-04-x64
        unique_name: yes
        ssh_key_ids: "{{ my_key.ssh_key.id }}"
    register: host0
  - add_host: 
      name: "{{ host0.droplet.ip_address }}"
      groups: host0
      ansible_ssh_private_key_file: keys/id_rsa
  - pause: seconds=20
```

There is unfortunately a lot going on here so I will try to summarize it in point form:

- The `- hosts: localhost` line tells Ansible we want to run the tasks of this play on localhost since we are only communicating with DigitalOcean's api at this point.

- In the first task `shell: cat keys/id_rsa.pub` we are utilizing the [shell module](http://docs.ansible.com/ansible/shell_module.html) to run a cat command on localhost machine. We are redirecting the output of the cat command onto a temporary [registered variable](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables) called `public_key` In summary, we are storing the public ssh key onto a temporary variable that can be used in the next step.

- In the `- name: Store public ssh key in digital ocean` and `- name: Create droplet` tasks, we are utilizing the [digital_ocean](http://docs.ansible.com/ansible/digital_ocean_module.html) module to communicate with DigitalOcean's api. The first task instructs DigitalOcean to store our public ssh key. The second task tells it to create a droplet. We catch the `api_token` using the [curly brackets](http://docs.ansible.com/ansible/playbooks_variables.html#hey-wait-a-yaml-gotcha) syntax.

- The final task utilizes [add_host](http://docs.ansible.com/ansible/add_host_module.html) module to store the IP address of the droplet as well as the corresponding private key necessary in order to ssh into it. We are storing the machine onto a hostname variable called `host0`.


In the next play, we are going to connect to `host0` and install python2 since it is a [requirement](http://docs.ansible.com/ansible/intro_installation.html#managed-node-requirements) in order to execute commands on a remote machine.


``` yml
- hosts: host0
  remote_user: root
  gather_facts: no
  tasks:
  - name: Install python2
    raw: apt-get -y install python-minimal
```
In summary we are:

- Using `host0` instead of `localhost` since our goal is to run tasks on the remote machine. Since we have already registered host0 with the appropriate IP address and ssh key, Ansible should be able to handle the connection automatically.  

- Connecting to the machine using `root` since that is the only available user

- Utilizing [raw](http://docs.ansible.com/ansible/raw_module.html) module to execute the installation of python2. Raw module is necessary because all other core modules won't work until python2 is installed.

Our last play is simple as we are going to setup/configure our HTTP server.

``` yml
- hosts: host0
  remote_user: root
  tasks:
- name: Install nginx
  apt: 
    name: nginx
    state: present
- name: Update nginx.conf
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
- name: Create /home/static directory
  file:
    path: /home/static
    state: directory
- name: Copy index.html
  copy:
    src: files/index.html
    dest: /home/static/index.html
- name: Restart nginx service
  service:
    name: nginx
    state: restarted
```

In summary we are:

- Connecting to the host machine again using `host0`

- Intalling Nginx using the [apt](http://docs.ansible.com/ansible/apt_module.html) module

- Copying the `nginx.conf` configuration file using the [copy](http://docs.ansible.com/ansible/copy_module.html)

- Creating the `/home/static` filepath and copying our index.html file onto the remote server

- Restarting the nginx webserver using the [service](http://docs.ansible.com/ansible/service_module.html) module

## Closing Thoughts

The most important realization that I want you to take out of this blog post is to have the mindset of automating everything. Whether it may be interfacing with a cloud provider to spin up/down virtual machines or installing/configuring applications -you will save a lot of time in the long run if you put that effort up front to focus on automation.

Getting used to managing servers with Ansible takes a little bit of effort as there is definitely a learning curve involved with the software. There are a ton of [core modules](http://docs.ansible.com/ansible/modules_by_category.html) available and [many extra modules](https://github.com/ansible/ansible-modules-extras) that are in active development. The possibilities are endless with what you can accomplish with it.

## Further Reading

If you are interested in learning Ansible at a more in-depth level, here are a list of resources I found to be helpful:

- https://www.ansible.com/webinars-training
- http://docs.ansible.com/ansible/index.html

