# Vagrant for Drupal Development
Drupalcamp Ottawa, February 23, 2013

## Alex Dergachev
Co-founder of [@evolvingweb](https://twitter.com/evolvingweb)
__@dergachev__ on [github](https://github.com/dergachev) and [twitter](https://twitter.com/dergachev)


------------------------------------


# Vagrant is...

* A wrapper on VBoxManage,
* A workflow,
* A way of life.


## Installing Vagrant

* Install virtualbox: https://www.virtualbox.org/wiki/Downloads
* Install Vagrant: 
http://downloads.vagrantup.com/

Simple installer on Windows/Mac/Linux


## Vagrant Docs

* http://vagrantup.com

* http://docs.vagrantup.com/v1/docs/
* http://docs.vagrantup.com/v1/docs/getting-started/




## Base boxes

Just a VM image with:

- "vagrant" user & ssh keys
- virtualbox guest additions
- installed ruby, chef, puppet

### Where to find base boxes

* Community boxes from http://www.vagrantbox.es/ 
* official builds have URLs like files.vagrantup.com/*.box

### Base box docs

* vagrantup.com doc on [base boxes](http://docs.vagrantup.com/v1/docs/base_boxes.html)
* Tutorial on [manually building a base box](https://github.com/fespinoza/checklist_and_guides/wiki/Creating-a-vagrant-base-box-for-ubuntu-12.04-32bit-server)
* [Vewee](https://github.com/jedi4ever/vewee) automates box creation.
  * [veewee tutorial](http://seletz.github.com/blog/2012/01/17/creating-vagrant-base-boxes-with-veewee/)


## Vagrant workflow

- vagrant box add precise64 http://files.vagrantup.com/precise64.box
- vagrant init
- vagrant up
  _ --no provision _
- vagrant reload
- vagrant provision
- vagrant destroy --force && vagrant up
- vagrant ssh
  _ vagrant ssh -p -- -l alex _
- ls /vagrant
  _ (inside the VM) _
- vagrant package
- vagrant snap
  _ requires vagrant-snap plugin_


## Vagrantfile

- config.vm.forward_port 80, 4567
- config.vm.box = "precise64"
- config.vm.share_folder "foo", "/guest/path", "/host/path", :nfs => true
- config.vm.network :hostonly, "10.11.12.13"
  "config.vm.network :bridged"
- config.vm.customize ["modifyvm", :id, "--memory", 1024]
- config.vm.provision :shell, :inline => "echo foo > /vagrant/test"
- support for multiple VMs


------------------------------------


# Provisioning

Installing software on the VM


## Shell provisioning

```
config.vm.provision :shell, :inline => "echo \"Europe/London\" | sudo tee /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata"
```


## Other provisioners

* Chef (and chef-client)
* Puppet
* CFengine


## Chef-solo and Vagrant

    config.vm.provision :chef_solo do |chef|
        chef.cookbooks_path = ["cookbooks", "~/company/cookbooks"]
        chef.add_recipe("apache")
        chef.add_recipe("php")
        chef.json = {
          :load_limit => 42,
          :chunky_bacon => true
        }
      end

More info:

* http://docs.vagrantup.com/v1/docs/provisioners/chef_solo.html
* http://fnichol.github.com/iterative_chef/


------------------------------------


# Ruby for Chef

Just enough not to freak out


## Methods
```
def double(num)
  num * 2
end
double 33 # => 66
```


## Ruby Strings

```
:bob == 'bob'.to_sym
%w{git apache2 mysql} == ['git', 'apache2', 'mysql']
"head #{variable} tail" == "head " +  variable + " tail"
```


## Ruby dictionaries

```
options = {
  :deploy_drupal => { 
    :apache_group => 'vagrant', 
    :sql_load_file => '/vagrant/db/fga.sql.gz',
    :source =>  "/vagrant/public/fga/www",
  },
  :mysql => {
    :server_root_password => "root",
  },
}
```



## DSL... wtf?

```
template "/etc/mysql/drupal-grants.sql" do
  path "/etc/mysql/drupal-grants.sql"
  source "grants.sql.erb"
  owner "root"
  group "root"
end
```

```
template("/etc/mysql/drupal-grants.sql", function() {  
  this.path("/etc/mysql/drupal-grants.sql");
  this.source("grants.sql.erb");
  this.owner("root");
  this.group("root");
});
```


------------------------------------


# Chef Basics

What can recipes do?


## Install a user

```
user "sam" do
     home "/home/sam"
     shell "/bin/zsh"
     comment "Sam loves DevOps"
     action :create
     password :$1$pfFfDG3M$22vsMsPnn93ZnuodI86Ec0
end
```


## Install a template

```
template '/var/www/sites/default/settings.php' do
  # action :create
  # action :create_if_missing
  source "settings.php.erb"
  variables ( {
    :user => 'root',
    :pass => node[:mysql]['root_password'],
    :name => 'drupalsite_v1',
  })
end
```


## Install packages

```
package "tar" do
  action :install
end
package %w{vim git}
php_pear "uploadprogress"
gem_package "syntax"
```



## Execute bash

```
execute "download drupal" do
  cwd '/var/www'
  command "drush dl drupal-7.20"
  creates '/var/www/drupal-7.20/index.php'
end
```


## Deploy from git

```
deploy "/my/deploy/dir" do
  repo "git@github.com/mycompany/project"
  user "www-data"
  group "www-data"
  revision "master"
  action :sync # default value, also :checkout
end
```


## defining and controlling service daemons

```
# runs /etc/init.d/apache2 (start|stop|restart), etc.
service "apache2" do
  supports :status => true, :restart => true, :reload => true
  action [ :enable, :start ]
end
```


## apache cookbook: webapp LWRP

```
web_app 'drupalcampottawa' do
  template "web_app.conf.erb"
  port 80
  server_name 'drupalcampottawa.com'
  docroot '/var/www/dcampottawa'
  notifies :restart, resources("service[apache2]"), :delayed
end
```



## Cookbook structure

![](http://dl-web.dropbox.com/u/29440342/screenshots/APTIAH-2013.2.22-16.51.png)

Adapted from [@opscode/getting-started](https://github.com/opscode-cookbooks/getting-started):

* recipes/
  * _default.rb_
  * _other_recipe.rb_
* attributes/
  * _default.rb_
* templates/
  * _httpd.conf.rb_
* _metadata.rb_
* _README.md_


## Cookbooks links

- http://wiki.opscode.com/display/chef/Chef+Basics
- http://docs.opscode.com/essentials_cookbooks.html
  - cookbook file structure
- https://github.com/opscode-cookbooks
  "officially supported cookbooks"


------------------------------------


# More Resources

Useful tools and extensions to Vagrant and Chef.


## Testing

* [Travis](https://travis-ci.org)
* [Food critic](http://acrmp.github.com/foodcritic/)
* [Minitest](https://github.com/calavera/minitest-chef-handler)


## Cookbook managers

These tools automatically download dependent cookbooks:

* [Librarian](https://github.com/applicationsonline/librarian)
* [Berkshelf](http://berkshelf.com/)

See this [berkshelf tutorial](http://vialstudios.com/guide-authoring-cookbooks.html)


## Vagrant Plugins

* Build your own base boxes: [@jedi4ever/veewee](https://github.com/jedi4ever/veewee)
* Manage virtualbox snapshots: [@jedi4ever/sahara](https://github.com/jedi4ever/sahara) and [@t9md/vagrant-snap](https://github.com/t9md/vagrant-snap)
* [@mosaicxm/vagrant-hostmaster](https://github.com/mosaicxm/vagrant-hostmaster) for managing /etc/hosts
* List of [Vagrant Plugins](https://github.com/mitchellh/vagrant/wiki/Available-Vagrant-Plugins)
* Docs on [writing Vagrant plugins](http://docs.vagrantup.com/v1/docs/extending/)


## More Vagrant info

* [@mitchellh/vagrant](https://github.com/mitchellh/vagrant)
* [Vagrant Docs](http://docs.vagrantup.com/v1/docs)
* IRC: [#vagrant on freenode](http://irc.lc/freenode/vagrant)


## Chef-server

* [Open-source tool](http://community.opscode.com/) to manage the state of all your servers that are provisioned by chef-client.
* developed by Opscode, who offer a [hosted version](http://www.opscode.com/hosted-chef/).


## Chef links

* http://wiki.opscode.com/display/chef/Home
* http://docs.opscode.com/
* [#ChefConf 2013](http://chefconf.opscode.com/) April 24-26 SFO
* [jtimperman's blog](http://jtimberman.housepub.org/) with advanced chef tips
* IRC: [#chef on freenode](http://irc.lc/freenode/chef)


## My chef contributions

* https://gist.github.com/3866825#vagrant-tutorial
  * how to provision users, set up SSH agent forwarding
* Redmine
  * [@dergachev/chef-redmine](https://github.com/dergachev/chef_redmine) cookbook
  * [@dergachev/vagrant_redmine](https://github.com/dergachev/vagrant_redmine) tutorial; see extensive [NOTES.md](https://github.com/dergachev/vagrant_redmine/blob/master/NOTES.md)
* [@dergachev/vagrant_drupal](https://github.com/dergachev/vagrant_drupal) - work in progress
* [@dergachev/chef_extend_lwrp]( https://github.com/dergachev/chef_extend_lwrp)
  * LWRP tutorial
* [@dergachev/chef_root_ssh_agent](https://github.com/dergachev/chef_root_ssh_agent) 
  * extends [@fnichol/chef-homesick](https://github.com/fnichol/chef-homesick)
* [@dergachev/chef_users](https://github.com/dergachev/chef_users)
  * fork of [@opscode-cookbooks/users](https://github.com/opscode-cookbooks/users)
* [@dergachev/chef_homesick_agent](https://github.com/dergachev/chef_homesick_agent)