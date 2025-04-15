---
layout: post
title: Getting temporal to really work
category: workflows, temporal, durability
---

# Getting Temporal to Really Work

[Temporal](https://temporal.io/) is what makes your code **crash proof**, that promises both reliability and durability when running a tasks. Basically it's a workflow engine that helps you to programmatically orchestrate steps to achieve a task, at same time provide features like pause-countinue, countinue from a failed state, wait infinitely(yeah almost) for something to get done etc. Actually to come to think of it, first two in themselves is a big thing. This is a self referencing notes as go from stage to stage.

## First Installation

I will choose an easy path of [docker-compose](https://github.com/temporalio/docker-compose/tree/b756b87), but use [vagrant](https://www.vagrantup.com/) with [libvirt](https://libvirt.org/) and use [Rockylinux](https://rockylinux.org/) to setup for setting up an isolated temporal environment.

```bash
$ mkdir -p ~/vags/temporal
$ cd ~/vags/temporal
$ cat > Vagrantfile <<VAGRANT
Vagrant.configure("2") do |config|
  config.vm.define :rock_vm do |rock_vm|
    # Temporal UI Port: 8080 -> 18080
    rock_vm.vm.network :forwarded_port, guest: 8080, host: 18080
    # Temporal GRPC Port: 7233 -> 17233
    rock_vm.vm.network :forwarded_port, guest: 7233, host: 17233
    rock_vm.vm.box = "generic/rocky9"
    rock_vm.vm.network "private_network", type: "dhcp"
    rock_vm.vm.hostname = "rock"
  end
end
VAGRANT
$ vagrant up
# ... # Some output of machine coming up
# login to the machine
$ vagran ssh
# do intial setup
[vagrant@temporal]$ sudo dnf upgrade
# ... some output
# install docker
[vagrant@temporal]$ sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
# ... some output
[vagrant@temporal]$ sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# ... some output
[vagrant@temporal]$ sudo systemctl --now enable docker
# Add user to docker group
[vagrant@temporal]$ sudo usermod -a -G docker vagrant
# logout and login after the above command
# ... some output
# install git and neovim
[vagrant@temporal]$ sudo dnf install -y neovim git
```

Now moving on to the setup of Temporal itself, it fairly hassel free. And as you can notice already have port forwards setup between the host and the guest, run by vagrant, we should be able to access the Temporal UI from the host.

```bash
[vagrant@temporal]$ mkdir -p ~/dc/temporal
[vagrant@temporal]$ cd ~/dc/temporal
[vagrant@temporal]$ git clone https://github.com/temporalio/docker-compose.git
[vagrant@temporal]$ cd  docker-compose
[vagrant@temporal]$ cat .env # has the all the versions that would be used to run the Temporal Installation
COMPOSE_PROJECT_NAME=temporal
CASSANDRA_VERSION=3.11.9
ELASTICSEARCH_VERSION=7.17.27
MYSQL_VERSION=8
TEMPORAL_VERSION=1.27.2
TEMPORAL_ADMINTOOLS_VERSION=1.27.2-tctl-1.18.2-cli-1.3.0
TEMPORAL_UI_VERSION=2.34.0
POSTGRESQL_VERSION=16
POSTGRES_PASSWORD=temporal
POSTGRES_USER=temporal
POSTGRES_DEFAULT_PORT=5432
OPENSEARCH_VERSION=2.5.0
[vagrant@temporal]$ docker compose -f docker-compose-postgres.yml up
# Above commands runs it in the foreground, I prefer to run in the background using `-d` at the end
```

The above set of commands will set up Temporal version `1.27.2`, running with Postgres `16` as database backend. I can now access the the Temporal UI from http://localhost:18080. Cool it works, now what? Before we take Temporal for a ride, I hardly know any Temporal concepts. So thats first!

## Temporal concepts

A **Workflow Execution** is the unit of work in Temporal. Wokflow Executions are executed concurrently, and they communicate with each other using message passing. Each Workflow Execution is exclusive access to its local state.

A **Temporal Application** is a set of Workflow Executions. A Temporal Application can consist of millions of Workflow Executions. A Temporal Application is a _reentrant process_(resumable, recoverable and reactive).

A **Temporal Failure** is Temporal's representation of various types of errors that occur in the system. For effective durability of the system, it very critical to understand and handle different _Tempral Failures_.

A user would write a **Temporal Workflow Function** in their favourite language, which would be the **Temporal Workflow Definition** that is eventually be the _Workflow Execution_ that is executed exactly once and to completion - in the presence of the arbitrary load or failures.

A **Event History** maintains the state of each step in a _Workflow Execution_. The case of failure in Workflow Execution an _Event History_ allows it resume from the last recorded event.

A **Temporal Service** and **Worker Processes** make up the **Temporal Platform**. The _Temporal Service_ is the supervisor, that executes your _Temporal Application_ inside of a _Work Process_, together completing a runtime to for your Temporal Application. The Temporal Service is at the heart of Temporal, which oversease all Temporal Executions, and all Temporal Applications communicate with Temporal Service as it is the state management system that persists all _Event History_ to the DB.

> Why is this so similar to Erlang's [GenServers](https://www.erlang.org/docs/24/man/gen_server) and [Supervision Trees](https://adoptingerlang.org/docs/development/supervision_trees/)?

Having those definitions handy but when talking about temporal with your colleagues you might not use those but rather two more important words **Workflow** and **Activities**.

A **Workflow** is a sequence of steps, that either get parallely, sequentially or mixedly executed, to perform a task. Think of a build and deployment pipeline. Temporal does not provide a no-code environment, rather it provides a Workflow-as-Code environment with flexibility and control.

An **Activity** is a step in your workflow, that does just one thing and one thing only. Think of building a docker image. When an _Activity_ fails temporal will retry it based on how it is defined.

These are enough concepts to get me started, as I now understand it...
- Use any supported Temporal SDK
- Create a _Temporal Application_
- The Application will define a _Temporal Workflow_-as-Code
- Each step of that code will be an _Activity_, which will be a _Temporal Execution_
- Run this _Temporal Application_, and it will connection to _Temporal Service_
- The _Temporal Service_ will create appropriate durable _Workers_ to execute the _Activities_

Cool. Now what. I guess I should start with **Hello World** of Temporal.

## Hello Temporal

I am wondering  what is a _Workflow_-as-code actually, as I am visual learner I will pick a simple example disect it. But before that would learn something about the `temporal` command. So let's fetch it and install it, and I am going to do it on my host machine.

```bash
pushd /tmp
# As of this wrinting following is the latest stable version
wget https://github.com/temporalio/cli/releases/download/v1.3.0/temporal_cli_1.3.0_linux_amd64.tar.gz
tar -xvf temporal_cli_1.3.0_linux_amd64.tar.gz
mv temporal ~/.local/bin/
popd
# Setup autocompletions and I use fish shell on my host
echo 'eval "$(temporal completion fish)"' >~/.config/fish/completions/temporal.fish
source ~/.config/fish/completions/temporal.fish
temporal # and tab-tab-tab...
```

So I can now talk to my installation of _Temporal Service_ running inside of my vagrant machine, but wait a minute its running on different port, `17233`, from the host. Now looking for an env variable, to setup the default Temporal Service lookup and found `TEMPORAL_ADDRESS`.

```bash
set -x TEMPORAL_ADDRESS localhost:17233
# On bash `export TEMPORAL_ADDRESS=localhost:17233`
temporal workflow list
# No output as we do not have anything as yet
```
