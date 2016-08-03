---
title: Introduction to Managing your DigitalOcean Droplets using Ansible
date: 2016-07-31 17:30:50
tags:
---

## The Problem

There are a series of steps required before you can deploy an application to the cloud. It usually looks like the following:

1. Start up a server using your cloud provider's interface. 
2. SSH into the machine to install all the required dependencies that your application server needs. 
3. Copy over your application build onto the server so that it can be deployed.


This process seems to work great for small pet projects, but imagine what happens if you need to restart your server from scratch? You would have to redo everything manually which could take a lot of time. It gets even worse if you have a large infrastructure to manage that is comprised of many machines.


## Ansible - Automate Everything

In this post, I'm going to show you how you can use [Ansible](https://www.ansible.com/) to automate everything. Ansible
is a configuration management software that can help you automate your IT infrastructure. It basically works by reading a yml file of instructions that describes exactly what actions you want to perform on a remote machine. It uses ssh to connect to a machine and then performs the actions defined in your yml file. 

The goal of this post is to help you start thinking about automation by showing a very basic example of a server setup. There are a ton of core concepts of Ansible that this blog post is not going to cover since it is out of scope. However, I feel this post is still valuable for those who are new to automation and want to quickly see what it is all about.


## Requirements

###### Sample Project

A [sample project](https://github.com/harneksidhu/blog-examples/tree/master/ansible-digital-ocean) with an Ansible infrastructure is provided for your convenience. This should be cloned onto your machine. The project utilizes a virtual machine to encapsulate Ansible as well as any required modules that it needs in order to operate. The virtual machine acts as the control machine from which all Ansible commands will be executed. We could have avoided the virtual machine and just installed Ansible on your host operating system. However, I found that having a consistent environment is beneficial especially if you want to be able to work on a variety of different operating systems.

###### VirtualBox/Vagrant

You will need to install [Virtualbox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.virtualbox.org/wiki/Downloads). Virtualbox provides you with the ability to run a virtual machine on your host operating system. You can think of Vagrant as a utility software that wraps around VirtualBox and provides you with the ability to provision a virtual machine in an automated way. Click [here](https://www.vagrantup.com/docs/provisioning/) if you would like to learn more about how it works.

###### DigitalOcean

Create an account DigitalOcean if you have not already. If you would like to have a $10 credit, you can use my referral link [here](https://m.do.co/c/385c5ec4be11). After creating your account, you will need to generate an [API token](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2) that Ansible can use in order to send requests to your account.

###### Create public/private ssh keys

The sample project requires a public/private ssh key pair. You could use `ssh-keygen` or any alternative software you feel comfortable creating key pairs with. Here is a good [resource](https://help.github.com/articles/generating-an-ssh-key/) that explains how to create the keys on Windows and Mac. 

## Organizing the Sample Project

With the sample project cloned onto your machine, you will need to perform the following steps in order for the demo to work:

1. Edit the `group_vars/all.yml` file and replace `<Enter_API_Token>` with your DigitalOcean API token.
2. Place the ssh public/private key pair under the `keys/` folder with the file names `id_rsa` and `id_rsa.pub`.

Your project directory should now look like this:

{% asset_img project-structure.png %}

After that, all you have to do is open your favourite terminal client and change the directory to where the sample project is cloned and run: 

``` sh
vagrant up
```

This should take a while as it downloads an Ubuntu image and installs all the required modules that Ansible needs in order to operate the demo.


## Run the demo

The sample project is a simple Ansible script that performs the following actions:

1. Instructs DigitalOcean to create a droplet
2. Installs Nginx HTTP server
3. Configures Nginx to serve a static web page

To see this in action, run the following commands:


``` sh
vagrant ssh
cd ansible-digital-ocean/
ansible-playbook setup.yml
```

All we did was connect to the local virtual machine which has Ansible installed using ssh. We then changed the directory on the machine to where our Ansible script (setup.yml) is stored. We finally used the `ansible-playbook setup.yml` command to fire the script. 

You should see an output similar to what is shown below:


{% asset_img playbook-output.png %}


If you head on over to your DigitalOcean dashboard, you should see a droplet that has been automatically created for you:

{% asset_img digital-ocean-dashboard.png %}


Since the server is running, you should be able to hit the IP address on your browser to see Nginx serving a basic HTML page.



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

- The `hosts: localhost` line tells Ansible we want to run the tasks of this play on localhost since we are only communicating with DigitalOcean's API at this point.

- In the first task (line 4) we are utilizing the [shell module](http://docs.ansible.com/ansible/shell_module.html) to run a cat command on the local virtual machine. We are redirecting the output of the cat command onto a temporary [registered variable](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables) called `public_key` In summary, we are storing the public ssh key onto a temporary variable that can be used in the next step.

- In the second third tasks (lines 7 and 16), we are utilizing the [digital_ocean](http://docs.ansible.com/ansible/digital_ocean_module.html) module to communicate with DigitalOcean's API. The first task instructs DigitalOcean to store our public ssh key. The second task tells it to create a droplet. We catch the `api_token` from the variables file (`group_vars/all.yml`) using the [curly brackets](http://docs.ansible.com/ansible/playbooks_variables.html#hey-wait-a-yaml-gotcha) syntax.

- The final task (line 28) utilizes the [add_host](http://docs.ansible.com/ansible/add_host_module.html) module to store the IP address of the droplet as well as the corresponding private key used to ssh into it. We are storing the machine onto a hostname variable called `host0`.


In the next play, we are going to connect to `host0` and install python2 since it is a [requirement](http://docs.ansible.com/ansible/intro_installation.html#managed-node-requirements) in order to execute commands using Ansible's core modules.


``` yml
- hosts: host0
  remote_user: root
  gather_facts: no
  tasks:
  - name: Install python2
    raw: apt-get -y install python-minimal
```
In summary we:

- Used `host0` instead of `localhost` since our goal is to run tasks on the remote machine. Ansible should be able to handle the connection automatically since we have already registered host0 with the appropriate IP address and ssh key.

- Connected to the machine using `root` since that is the only user account available on the machine.

- Utilized the [raw](http://docs.ansible.com/ansible/raw_module.html) module to execute the installation of python2. Raw module is used here because the core modules will not work until python2 is installed.

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

In summary we:

- Connected to the host machine again using `host0`.

- Installed Nginx using the [apt](http://docs.ansible.com/ansible/apt_module.html) module.

- Copied the `nginx.conf` configuration file using the [copy](http://docs.ansible.com/ansible/copy_module.html) module.

- Created the `/home/static` file path and copying our index.html file onto the remote server.

- Restarted the Nginx webserver using the [service](http://docs.ansible.com/ansible/service_module.html) module.

## Closing Thoughts

The most important lesson that I want you to take out of this blog post is to have the mindset of **automating everything**. You will save time in the long run if you can put some effort up front and automate your infrastructure. You might think that the effort may not be worth it for basic deployments like the one used in this post but consider the following scenarios:

- You have new requirements which means you need to update the server. You could do this manually but making modifications to an existing system can lead to issues like forgetting to delete something you no longer need. We can use Ansible to execute the `destroy.yml` playbook and delete the machine entirely. Then we can modify/run the `setup.yml` playbook with the updated specifications and bring up a new server.

- You have to move your server to AWS. We can accomplish this by replacing the tasks in `setup.yml` that uses DigitalOcean modules with the corresponding [AWS](http://docs.ansible.com/ansible/list_of_cloud_modules.html#amazon) modules. Everything else basically remains the same making the migration pretty easy.

Getting used to managing servers with Ansible definitely takes some effort since there is a learning curve involved. Once you overcome that hurdle, you should be able to see Ansible as a useful tool that you can use whenever you need to work with a production machine. 

This blog post covers the bare minimum of what Ansible can do. There are a ton of [core modules](http://docs.ansible.com/ansible/modules_by_category.html) available and [many extra modules](https://github.com/ansible/ansible-modules-extras) that are in active development. The possibilities are endless with what you can accomplish with it.

## Further Reading

Learning Ansible at a more in-depth level takes some time but the investment is worth it. Here are some resources that I found that have helped me: 

- https://www.ansible.com/webinars-training
- http://docs.ansible.com/ansible/index.html

