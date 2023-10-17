# Terraform Up & Running [Writing Infrastructure as Code]

```bash
$ terraform init
$ terraform apply
```

DevOps practices:  (e.g., automated testing, Continuous Integration, Continuous Delivery) and tooling (e.g., Docker, Chef, Puppet).

## Questions You Should Be Able to Answer 

* What are the differences between **configuration management**, **orchestration**, **provisioning**, and **server templating**?
* When should you use Terraform, Chef, Ansible, Puppet, Pulumi, CloudFormation, Docker, Packer, or Kubernetes?
* How do you create reusable Terraform modules?
* How do you securely manage secrets when working with Terraform?
* How do you write Terraform code that’s reliable enough for production usage?
* How do you test your Terraform code?
* How do you make Terraform a part of your automated deployment process?
* What are the best practices for using Terraform as a team?


There is one area where it’s lacking: **maturity**.

Terraform has become wildly popular, but it’s still a relatively new technology, and despite its popularity, it’s still difficult to find books, blog posts, or experts to help you become proficient with the tool. The official Terraform documentation does a good job of introducing the basic syntax and features, but it includes little information on idiomatic patterns, best practices, testing, reusability, or team workflows. It’s like trying to become fluent in French by studying only the vocabulary but not any of the grammar or idioms.

*  how to build Terraform modules for production; small modules; composable modules; testable modules; releasable modules; Terraform Registry; variable validation; versioning Terraform, Terraform providers, Terraform modules, and Terragrunt; Terraform escape hatches.
*  Manual tests for Terraform code; sandbox environments and cleanup; automated tests for Terraform code; Terratest; unit tests; integration tests; end-to-end tests; dependency injection; running tests in parallel; test stages; retries; the test pyramid; static analysis; plan testing; server testing.


# Chapter 1. Why Terraform

Both Dev and Ops spend most of their time working on software, and the distinction between the two teams is blurring. It might still make sense to have a separate Dev team responsible for the application code and an Ops team responsible for the operational code, but it’s clear that Dev and Ops need to work more closely together. This is where the DevOps movement comes from.

The goal of DevOps is to make software delivery vastly more efficient.

Instead of multiday merge nightmares, you integrate code continuously and always keep it in a deployable state. Instead of deploying code once per month, you can deploy code dozens of times per day, or even after every single commit. And instead of constant outages and downtime, you build resilient, self-healing systems and use monitoring and alerting to catch problems that can’t be resolved automatically.


The results from companies that have undergone DevOps transformations are astounding. For example, Nordstrom found that after applying DevOps practices to its organization, it was able to increase the number of features it delivered per month by 100%, reduce defects by 50%, reduce lead times (the time from coming up with an idea to running code in production) by 60%, and reduce the number of production incidents by 60% to 90%. After HP’s LaserJet Firmware division began using DevOps practices, the amount of time its developers spent on developing new features went from 5% to 40%, and overall development costs were reduced by 40%. Etsy used DevOps practices to go from stressful, infrequent deployments that caused numerous outages to deploying 25 to 50 times per day, with far fewer outages.1

There are four core values in the DevOps movement: **culture**, **automation**, **measurement**, and **sharing** (sometimes abbreviated as the acronym CAMS).

The goal is to automate as much of the software delivery process as possible. That means that you manage your infrastructure not by clicking around a web page or manually executing shell commands, but through code. This is a concept that is typically called **infrastructure as code**.


## What Is Infrastructure as Code?

The idea behind infrastructure as code (IaC) is that you write and execute code to define, deploy, update, and destroy your infrastructure. This represents an important shift in mindset in which you treat all aspects of operations as software—even those aspects that represent hardware (e.g., setting up physical servers). In fact, a key insight of DevOps is that you can manage almost everything in code, including servers, databases, networks, logfiles, application configuration, documentation, automated tests, deployment processes, and so on.

There are five broad categories of IaC tools:

* Ad hoc scripts
* Configuration management tools
* Server templating tools
* Orchestration tools
* Provisioning tools

### Ad Hoc Scripts

The most straightforward approach to automating anything is to write an *ad hoc* script. You take whatever task you were doing manually, break it down into discrete steps, use your favorite scripting language (e.g., Bash, Ruby, Python) to define each of those steps in code, and execute that script on your server.

The great thing about ad hoc scripts is that you can use popular, general-purpose programming languages, and you can write the code however you want. The terrible thing about ad hoc scripts is that you can use popular, general-purpose programming languages, and you can write the code however you want.

Whereas tools that are purpose-built for IaC provide concise APIs for accomplishing complicated tasks, if you’re using a general-purpose programming language, you need to write completely custom code for every task. Moreover, tools designed for IaC usually enforce a particular structure for your code, whereas with a general-purpose programming language, each developer will use their own style and do something different. Neither of these problems is a big deal for an eight-line script that installs Apache, but it gets messy if you try to use ad hoc scripts to manage dozens of servers, databases, load balancers, network configurations, and so on.

If you’ve ever had to maintain a large repository of Bash scripts, you know that it almost always devolves into a mess of unmaintainable spaghetti code. Ad hoc scripts are great for small, one-off tasks, but if you’re going to be managing all of your infrastructure as code, then you should use an IaC tool that is purpose-built for the job.


### Configuration Management Tools

Chef, Puppet, and Ansible are all *configuration management tools*, which means that they are designed to install and manage software on existing servers. For example, here is an Ansible role called `web-server.yml` that configures the same Apache web server as the `setup-webserver.sh` script:

```yaml
- name: Update the apt-get cache
  apt:
    update_cache: yes

- name: Install PHP
  apt:
    name: php

- name: Install Apache
  apt:
    name: apache2

- name: Copy the code from the repository
  git: repo=https://github.com/brikis98/php-app.git dest=/var/www/html/app

- name: Start Apache
  service: name=apache2 state=started enabled=yes
```

Advantages:

#### Coding conventions
Ansible enforces a consistent, predictable structure, including documentation, file layout, clearly named parameters, secrets management, and so on. While every developer organizes their ad hoc scripts in a different way, most configuration management tools come with a set of conventions that makes it easier to navigate the code.

####  Idempotence
Writing an ad hoc script that works once isn’t too difficult; writing an ad hoc script that works correctly even if you run it over and over again is much harder. Every time you go to create a folder in your script, you need to remember to check whether that folder already exists; every time you add a line of configuration to a file, you need to check that line doesn’t already exist; every time you want to run an app, you need to check that the app isn’t already running.

Code that works correctly no matter how many times you run it is called idempotent code. To make the Bash script from the previous section idempotent, you’d need to add many lines of code, including lots of if-statements. Most Ansible functions, on the other hand, are idempotent by default. For example, the web-server.yml Ansible role will install Apache only if it isn’t installed already and will try to start the Apache web server only if it isn’t running already.

#### Distribution

Ad hoc scripts are designed to run on a single, local machine. Ansible and other configuration management tools are designed specifically for managing large numbers of remote servers.

For example, to apply the web-server.yml role to five servers, you first create a file called hosts that contains the IP addresses of those servers:

```bash
[webservers]
11.11.11.11
11.11.11.12
11.11.11.13
11.11.11.14
11.11.11.15
```

Next, you define the following Ansible playbook:

```bash
- hosts: webservers
  roles:
  - webserver
```

Finally, you execute the playbook as follows:

```yaml
ansible-playbook playbook.yml
```

This instructs Ansible to configure all five servers in parallel. Alternatively, by setting a parameter called `serial`` in the playbook, you can do a rolling deployment, which updates the servers in batches. For example, setting `serial` to `2` directs Ansible to update two of the servers at a time, until all five are done. Duplicating any of this logic in an ad hoc script would take dozens or even hundreds of lines of code.

### Server Templating Tools

An alternative to configuration management that has been growing in popularity recently are server templating tools such as Docker, Packer, and Vagrant. Instead of launching a bunch of servers and configuring them by running the same code on each one, the idea behind server templating tools is to create an image of a server that captures a fully self-contained “snapshot” of the operating system (OS), the software, the files, and all other relevant details. You can then use some other IaC tool to install that image on all of your servers.

You can use a server templating tool like Packer to create a self-contained image of a server. You can then use other tools, such as Ansible, to install that image across all of your servers

There are two broad categories of tools for working with images:

#### Virtual machines

A virtual machine (VM) emulates an entire computer system, including the hardware. You run a hypervisor, such as VMware, VirtualBox, or Parallels, to virtualize (i.e., simulate) the underlying CPU, memory, hard drive, and networking.

The benefit of this is that any VM image that you run on top of the hypervisor can see only the virtualized hardware, so it’s fully isolated from the host machine and any other VM images, and it will run exactly the same way in all environments (e.g., your computer, a QA server, a production server). The drawback is that virtualizing all this hardware and running a totally separate OS for each VM incurs a lot of overhead in terms of CPU usage, memory usage, and startup time. You can define VM images as code using tools such as Packer and Vagrant.

#### Containers

A container emulates the user space of an OS.2 You run a container engine, such as Docker, CoreOS rkt, or cri-o, to create isolated processes, memory, mount points, and networking.

The benefit of this is that any container you run on top of the container engine can see only its own user space, so it’s isolated from the host machine and other containers and will run exactly the same way in all environments (your computer, a QA server, a production server, etc.). The drawback is that all of the containers running on a single server share that server’s OS kernel and hardware, so it’s much more difficult to achieve the level of isolation and security you get with a VM.3 However, because the kernel and hardware are shared, your containers can boot up in milliseconds and have virtually no CPU or memory overhead. You can define container images as code using tools such as Docker and CoreOS rkt; 