# Vagrant Setup

This tutorial guides you through creating your first Vagrant project.

We start with a generic Ubuntu VM, and use the Chef provisioning tool to:
* install packages for vim, git
* create user accounts, as specified in included JSON config files
* install specified user dotfiles (.bashrc, .vimrc, etc) from a git repository

Afterwards, we'll see how easy it is to package our newly provisioned VM
environment into a new "Vagrant box", which can be instantly deployed for new
projects, and shared with others.


## Install Vagrant

For a backgrond on Vagrant, see:
* http://vagrantup.com/v1/docs/getting-started/index.html
* http://vagrantup.com/v1/docs/provisioners/chef_solo.html

Download and install VirtualBox from https://www.virtualbox.org/wiki/Downloads
Download and install Vagrant from http://downloads.vagrantup.com

## Setup up a Vagrant project:

Download the Precise Pangolin Ubuntu 12.04 base vagrant box, see http://vagrantbox.es/

```bash
    vagrant box add precise64 http://files.vagrantup.com/precise64.box # 323MB, faster download
```

Initialize a new Vagrant project in ~/code/vagrant-tutorial, based on the
precise64 box we just installed: 

```bash
    mkdir -p ~/code/vagrant-tutorial/ & cd ~/code/vagrant-tutorial/
    vagrant init precise64 # creates a default Vagrantfile in the current directory
```

## Creating a VM

You can now ask Vagrant to start up a VM as configured by the default Vagrantfile:

```
  vagrant up 
```

The VM is now running in Virtualbox. You can ssh into it (no password required)
as follows: 

```
  vagrant ssh # ssh in to the VM
```

Note that on the new VM, /vagrant is a shared directory linked with
~/code/vagrant-tutorial

```
  ls /vagrant #shared folder mounted to your project root
```

Soon, you'll start making modifications to the Vagrantfile. There are three ways
to ask Vagrant to rebuild the VM.

```bash
  # Fastest method: re-runs the provisioner (eg chef-solo) without stopping the VM.
  vagrant provision 

  # Restarts VM, ru-provisions. Use this if you changed virtualbox settings (eg shared folders)
  vagrant reload 

  # Destroys the active VM, and rebuilds from the base box.
  # Slow, but guarantees stability.
  vagrant destroy --force && vagrant up # deletes the VM and rebuilds 
```

## Provision VM customizations using Chef

The main benefit of Vagrant is to easily provision VMs with your
customizations, which generally include the following: 

* installing packages (`apt-get install vim`)
* updating configuration files (`/etc/apache2/httpd.conf`)
* setting up user accounts
* running various scripts

Vagrant calls this process "provisioning", and supports several popular tools including Puppet and Chef. 

This tutorial will use Chef-solo, a single-server version of Chef. Chef calls provisioning scripts "recipes", and related recipes are grouped into "cookbooks". (Get the idea?)

For this tutorial, we will use chef-solo to provision the following Chef cookbooks:

* vim: basically runs 'sudo apt-get install vim'
* git: basically runs 'sudo apt-get install git'
* users: creates user accounts from JSON configuration files
* homesick: installs public git repositories for each user's dotfiles (eg .vimrc, .bashrc) 

The following additional cookbooks are required to support dotfiles
repositories requiring SSH key authentication via agent forwarding:

* homesick_agent
* extend_lwrp
* root_ssh_agent
* ssh_known_hosts

For more information about how the custom cookbooks work (and for my notes about learning chef), see the following writeups:

* https://github.com/dergachev/chef_homesick_agent#readme 
* https://github.com/dergachev/chef_users#readme 
* https://github.com/dergachev/chef_root_ssh_agent#readme
* https://github.com/dergachev/chef_extend_lwrp#readme

### Install and configure cookbooks

Create sub-directories for cookbooks (chef deployment scripts) and databags (JSON config files):

```bash
  mkdir ~/code/vagrant-tutorial/site-cookbooks
  mkdir ~/code/vagrant-tutorial/cookbooks
  mkdir ~/code/vagrant-tutorial/databags
```

Install the following community-maintained cookbooks to `cookbooks`:

```bash
  cd ~/code/vagrant-tutorial/cookbooks
  git clone git://github.com/fnichol/chef-homesick.git homesick
  git clone git://github.com/opscode-cookbooks/ssh_known_hosts.git
  git clone git://github.com/opscode-cookbooks/vim.git
  git clone git://github.com/opscode-cookbooks/git.git
```

In the course of creating this tutorial, I created the following custom ("site-specific") cookbooks. Install them into `site-cookbooks`:

```bash
  cd ~/code/vagrant-tutorial/site-cookbooks
  git clone git://github.com/dergachev/chef_homesick_agent.git homesick_agent
  git clone git://github.com/dergachev/chef_root_ssh_agent.git root_ssh_agent
  git clone git://github.com/dergachev/chef_users.git users
  git clone git://github.com/dergachev/chef_extend_lwrp.git extend_lwrp
```

When installing cookbooks, be mindful that cookbooks folder names must match
the cookbook name.

* Wrong: `git clone git://github.com/dergachev/chef_users.git`
* Correct: `clone git://github.com/dergachev/chef_users.git users`

### Install databags 

The `users`, `homesick`, and `ssh_known_hosts` cookbooks require JSON
configuration files to be placed in the following sub-directories:

```bash
  mkdir -p ~/code/vagrant-tutorial/databags/users
  mkdir -p ~/code/vagrant-tutorial/databags/ssh_known_hosts
```

For each user to be created, you'll need to provide a config file at
`databags/users/USERNAME.json`. If a user has a dotfiles directory which should
be installed (homesick calls them castles), you'll need to insert an addtional 'homesick_castles' property.

### Sample databags/users/testuser.json

For example, save the following into `databags/users/testuser.json`, taking
care to modify the following fields: 

* `id` : the username. Should be the same as the json filename.
* `password`: hashed password, generated via `openssl passwd -1 "password123"` 
  * for the security conscious, run `openssl passwd -1` and type a password in
* `ssh_keys`: the contents of your SSH public key file
  * If you've already set one up, you can probably get it by running `cat ~/.ssh/id_rsa.pub`.  
  * For help setting one up, see https://help.github.com/articles/generating-ssh-keys
  * For help with SSH agent forwarding, see https://help.github.com/articles/using-ssh-agent-forwarding
* `homesick_castles`: your dotfiles git repository URL to 'git clone'
  * the dotfiles repository must be compatible with homesick, see https://github.com/technicalpickles/homesick#readme
  * For read-only (HTTP) clone, use "git://github.com/technicalpickles/dotpickles.git"
  * For read-write (SSH) clone, use "git@github.com:technicalpickles/dotpickles.git".

```json
{
  "groups": [
    "sysadmin"
  ],
  "comment": "Test User Name",
  "password": "$6$BcvQ4/W1iMlP9S33$k4RmfftqRi1I5T.z113L1VrXX0K78Uwii8Ot4WC1p74m2agZHYqfp9eNYG10B6adrQIEJ4jQyagJiMt7q9MiF.",
  "ssh_keys": [
    "ssh-rsa AAA456...uvw== testuser@testdomain.com"
  ],
  "id": "alex",
  "homesick_castles": [
    {
      "name": "dotfiles",
      "source": "git://github.com/technicalpickles/dotpickles.git"
    }
  ],
  "shell": "/bin/bash",
  "email": "testuser@testdomain.com"
}
```

For more information about the respective databag formats, see
* https://github.com/dergachev/chef_users#usage
* https://github.com/fnichol/chef-homesick/tree/#-usage

### Sample databags/ssh_known_hosts/github.json

If any `homesick_castle` entry contains a SSH git URL (necessary for read-write
access, generally of the form
`git@github.com:technicalpickles/dotpickles.git`), you will need to add that
host key signature into `databags/ssh_known_hosts/hostname.json`.

For example, if your git repository server is github.com, create
`databags/ssh_known_hosts/github.json` as follows:

```json
{
"id": "github",
"fqdn": "github.com",
"rsa": "AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==",
"ipaddress": "207.97.227.239"
}
```

If your git host is `git.mycompany.com`, you should create `git-mycompany.json`
and populate it with values derived from `cat ~/.ssh/known_hosts | grep git
mycomapny.com`. For more info, see 
https://github.com/dergachev/chef_homesick_agent#install-ssh_known_hosts-cookbook

### Install Vagrantfile

With this done, you need specify which cookbooks and recipes Vagrant
should deploy, via the chef-solo provisioner. To do this, replace the default
Vagrantfile (`~/code/vagrant-tutorial/Vagrantfile`) with the following:

```ruby
Vagrant::Config.run do |config|
  config.vm.box = "precise64"

  # Necessary for homesick_agent::data_bag
  config.ssh.forward_agent = true

  config.vm.provision :chef_solo do |chef|

    # contains "users" and "ssh_known_hosts" databags
    chef.data_bags_path = "databags"

    chef.cookbooks_path = ["cookbooks", "site-cookbooks"]

    # stuff that should be in base box
    chef.add_recipe "vim"
    chef.add_recipe "git"
    
    # setup users (from data_bags/users/*.json)
    chef.add_recipe "users::ruby_shadow" # necessary for password shadow support
    chef.add_recipe "users::sysadmins" # creates users and sysadmin group
    chef.add_recipe "users::sysadmin_sudo" # adds %sysadmin group to sudoers

    # homesick_agent and its dependencies
    chef.add_recipe "root_ssh_agent::ppid" # maintains agent during 'sudo su root'
    chef.add_recipe "ssh_known_hosts" # populates /etc/ssh/ssh_known_hosts from data_bags/ssh_known_hosts/*.json
    chef.add_recipe "homesick_agent::data_bag" # includes homesick::data_bag
    
    # instruct "homesick::data_bag" to install dotfiles for the user 'testuser'
    chef.json = {
       :users => ['testuser']
    }

    # chef.log_level = :debug

  end
end
```

Be sure to modify the `chef.json[:users]` property to reflect the user(s)
you're configuring in your databags.

## Launch the VM

If you did everything right, you are ready to start the VM!

```bash 
  vagrant destroy --force && vagrant up
```

It'll take a minute to start the VM, and then several more to run chef-solo to
install all the cookbooks and recipes specified in `Vagrantfile`. 

It's entirely possible that something goes wrong. Pay close attention to chef
errors, to figure out which installation step wasn't performed correctly. If
you get stuck, consider uncommenting the following in Vagrantfile:

```ruby
  chef.log_level = :debug
```

As you tweak things, you'll generally need to re-run `vagrant provision` to see
if the errors go away. Keep in mind that chef will halt on the first error, so
often you'll fix one error only to stumble upon the next one. In some cases,
you'll need to run `vagrant destroy --force ; vagrant up`, to really start
from a clean slate.

Good luck!


## Connect to VM

Once you've resolved any chef errors After doing that, ssh to the VM as as one of the newly-created users 

```bash
  vagrant ssh -p -- -l testuser   #replace 'testuser' with your username
  ls -al ~ # dotfiles should be symlinked to ~/.homesick
```

Note that vagrant also port forwards `localhost:8080` to `vagrant:80`. So if
you have a web server setup on your VM, you will be able to acess it at
`http://127.0.0.1:8080`.

## Package a customized Vagrant box

At this point, we can package our customizations to the `precise64` base
Vagrant box into `precise64-customized`, to use as the box for future projects.
These steps follow the Vagrant packaging tutorial:
http://vagrantup.com/v1/docs/getting-started/packaging.html

Note: before trying this for the second time, you'll need to remove previously
exported boxes as follows:
```bash
  cd ~/code/vagrant-tutorial
  rm ~/code/vagrant-tutorial/package.box
  vagrant box remove precise64-customized
```

First, create this file: `~/code/vagrant-tutorial/Vagrantfile.pkg` as follows:

```ruby
Vagrant::Config.run do |config|
  config.vm.forward_port 80, 8080
  config.ssh.forward_agent = true # important for recipe[homesick_agent::data_bag]
end
```

Now export the currently running VM, bundled with the above Vagrantfile.pkg: 

```bash
  vagrant package --vagrantfile Vagrantfile.pkg #creates ~/code/vagrant-tutorial/package.box
```

Now add it as `precise64-customized` to your user's set of available boxes (stored in `~/.vagrant.d/boxes/`):

```bash
  vagrant box add precise64-customized ~/code/vagrant-tutorial/package.box
```

If that worked, you're now ready to test it with a new project: 
   
```bash
  mkdir -p ~/code/new-vagrant-project ; cd ~/code/new-vagrant-project
  # generate a Vagrantfile that includes config.vm.box="precise64-customized"
  vagrant init precise64-customized 
  vagrant up
```

Now test that everything works correctly (including user deployment):

```bash
  cd ~/code/new-vagrant-project
  vagrant ssh -p -- -l testuser # replace 'testuser' with your username
  ls -al ~    # various dotfiles should be symlinked to ~/.homesick
```

That's it! (Hopefully).

For info about bundling Vagrantfiles and Vagrantfile overloading, see
http://vagrantup.com/v1/docs/vagrantfile.html#vagrantfile_load_order

## Building a base box from scratch

If you want to build a base box VM from scratch, see the following:

* http://vagrantup.com/v1/docs/base_boxes.html
* https://github.com/fespinoza/checklist_and_guides/wiki/Creating-a-vagrant-base-box-for-ubuntu-12.04-32bit-server
* http://briceno.mx/2012/10/easy-guide-to-create-a-vagrant-box-from-virtualbox/

Also check out VeeWee, a tool to create a Vagrant base box from a
distribution ISO image:

* https://github.com/jedi4ever/veewee 
* http://seletz.github.com/blog/2012/01/17/creating-vagrant-base-boxes-with-veewee/

