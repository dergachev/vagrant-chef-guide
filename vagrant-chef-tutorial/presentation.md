# Vagrant & Chef
             ... for Drupal developers

__Alex Dergachev__ | Co-founder of [Evolving Web](http://evolvingweb.ca)

[@dergachev](https://github.com/dergachev) on github | [@dergachev](https://twitter.com/dergachev) on twitter





---
# Vagrant is...

* A wrapper on VBoxManage,
* A workflow,
* A way of life.


---
## Installing Vagrant

Run these GUI installers:

* [Download Virtualbox](https://www.virtualbox.org/wiki/Downloads)
* [Download Vagrant](http://downloads.vagrantup.com/)

Both are simple, reliable, and work on Windows/Mac/Linux


---
## Vagrant Docs

* [vagrantup.com](http://vagrantup.com)
* [docs.vagrantup.com/v1/docs/](http://docs.vagrantup.com/v1/docs/)
* [docs.vagrantup.com/v1/docs/getting-started](http://docs.vagrantup.com/v1/docs/getting-started)



---
## Base boxes

Just a VM image with:

- "vagrant" user & ssh keys
- virtualbox guest additions
- installed ruby, chef, puppet

---
### Where to find base boxes

* Community boxes from http://www.vagrantbox.es/ 
* official builds have URLs like files.vagrantup.com/*.box

---
### Base box docs

* vagrantup.com doc on [base boxes](http://docs.vagrantup.com/v1/docs/base_boxes.html)
* Tutorial on [manually building a base box](https://github.com/fespinoza/checklist_and_guides/wiki/Creating-a-vagrant-base-box-for-ubuntu-12.04-32bit-server)
* [Vewee](https://github.com/jedi4ever/vewee) automates box creation.
  * [veewee tutorial](http://seletz.github.com/blog/2012/01/17/creating-vagrant-base-boxes-with-veewee/)


---
## Vagrant workflow

```bash
vagrant box add precise64
  http://files.vagrantup.com/precise64.box
vagrant init
vagrant up # --no provision
vagrant reload
vagrant provision
vagrant destroy --force && vagrant up
vagrant ssh  # -p -- -l alex
ls /vagrant # inside the VM
vagrant package # outputs ./package.box
vagrant snap take # requires vagrant-snap plugin
```


---
## Vagrantfile config

```ruby
config.vm.forward_port 80, 4567
config.vm.box = "precise64"
config.vm.share_folder "foo", "/guest/path", 
  "/host/path", :nfs => true
config.vm.network :hostonly, "10.11.12.13"
 # config.vm.network :bridged
config.vm.customize 
  ["modifyvm", :id, "--memory", 1024]
config.vm.provision :shell, 
  :inline => "echo foo > /vagrant/test"
 # support for multiple VMs
```





---
# Provisioning

Installing software on the VM


---
## Supported provisioners

* shell
* chef-solo
* chef
* Puppet


---
## Shell provisioner

```ruby
config.vm.provision :shell, 
  :inline => "apt-get -y update"

config.vm.provision :shell, 
  :inline => "sudo apt-get -y vim git"
```


---
## Chef-solo provisioner

```ruby
config.vm.provision :chef_solo do |chef|
  chef.cookbooks_path = 
    ["cookbooks", "~/company/cookbooks"]
  chef.add_recipe("apache")
  chef.add_recipe("php")
  chef.json = {
    :load_limit => 42,
    :chunky_bacon => true
  }
end
```

See [Vagrant chef_solo docs]( http://docs.vagrantup.com/v1/docs/provisioners/chef_solo.html) and [iterative chef](http://fnichol.github.com/iterative_chef/) tutorial.





---
# Ruby for Chef

Just enough not to freak out


---
## Methods

```ruby
def double(num)
  num * 2
end

double 33 # => 66
```


---
## Ruby Strings

```ruby
:bob == 'bob'.to_sym

%w{git vim mysql} == ['git', 'vim', 'mysql']

"head #{var} tail" == "head "+var+" tail"
```


---
## Ruby dictionaries

```ruby
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



---
## DSL... wtf?

```ruby
template "/etc/mysql/grants.sql" do
  source "grants.sql.erb"
  owner "root"
  group "root"
end
```

```javascript
template("/etc/mysql/grants.sql", function() {  
  this.source("grants.sql.erb");
  this.owner("root");
  this.group("root");
});
```





---
# Writing a cookbook




---
## Cookbooks and recipes 

A __cookbooks__ contains __recipes__, which are _very simple_ ruby scripts that install software or otherwise configure the VM. 

Recipes read data from __attributes__, and call built-in or custom __resources__, which are helper methods that do all the work. 


---
## Cookbook structure

![](http://dl-web.dropbox.com/u/29440342/screenshots/APTIAH-2013.2.22-16.51.png)



---
## Example Cookbook

Adapted from [@opscode/getting-started](https://github.com/opscode-cookbooks/getting-started):

* _recipes/_
  * default.rb
  * other_recipe.rb
* _attributes/_
  * default.rb
* _templates/_
  * httpd.conf.erb
* metadata.rb
* README.md


---
## Resources

* __Recipes__ are ruby scripts, but with minimal code.
* Recipes specify which system configuration __Resources__ chef should execute
* Resource implementations (called __Providers__) actually do the work.
* Simplifying, recipes contain "API calls" to resources, with optional arguments.


---
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


---
## Install packages

```
package "tar" do
  action :install
end

package %w{vim git}

php_pear "uploadprogress"

gem_package "syntax"
```



---
## Execute bash

```
execute "download drupal" do
  cwd '/var/www'
  command "drush dl drupal-7.20"
  creates '/var/www/drupal-7.20/index.php'
end
```

__creates__ tells chef not to re-execute the resource if index.php is already in place.


---
## Deploy from git

```
deploy "/var/www" do 
  repo "git@github.com/mycompany/project"
  user "www-data"
  group "www-data"
  revision "master"
  action :sync # default value, also :checkout
end
```


---
## Controlling services

```
 # runs /etc/init.d/apache2 start|stop|...
service "apache2" do
  supports :status => true, 
    :restart => true, :reload => true
  action [ :enable, :start ]
end
```


---
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


---
## Add a vhost config

```ruby
web_app 'drupalcampottawa' do
  template "web_app.conf.erb"
  port 80
  server_name 'drupalcampottawa.com'
  docroot '/var/www/dcampottawa'
  notifies :restart, 
    resources("service[apache2]"), 
    :delayed
end
```

__web_app__ is custom resource defined in the apache2 cookbook


---
## Example recipe

Putting it all together, you should be able to understand this [example recipe](https://github.com/fespinoza/checklist_and_guides/blob/master/chef%20solo%20guide/cookbooks/main/recipes/default.rb)


---
## More about recipes and resources

* [docs.opscode.com/chef/resources.html](http://docs.opscode.com/chef/resources.html) (READ ALL OF THIS)
* [wiki.opscode.com/display/chef/Chef+Basics](http://wiki.opscode.com/display/chef/Chef+Basics)
* [docs.opscode.com/essentials_cookbooks.html](http://docs.opscode.com/essentials_cookbooks.html)
* [wiki.opscode.com/display/chef/Resources+and+Providers](http://wiki.opscode.com/display/chef/Resources+and+Providers)







---
# Additional stuff

Useful tools and extensions to Vagrant and Chef.


---
## Testing

* [Travis](https://travis-ci.org)
* [Food critic](http://acrmp.github.com/foodcritic/)
* [Minitest](https://github.com/calavera/minitest-chef-handler)


---
## Cookbook managers

These tools automatically download dependent cookbooks:

* [Librarian](https://github.com/applicationsonline/librarian)
* [Berkshelf](http://berkshelf.com/)

See this [berkshelf tutorial](http://vialstudios.com/guide-authoring-cookbooks.html)


---
## Vagrant Plugins

* Build your own base boxes: [@jedi4ever/veewee](https://github.com/jedi4ever/veewee)
* Manage virtualbox snapshots: [@jedi4ever/sahara](https://github.com/jedi4ever/sahara) and [@t9md/vagrant-snap](https://github.com/t9md/vagrant-snap)
* [@mosaicxm/vagrant-hostmaster](https://github.com/mosaicxm/vagrant-hostmaster) for managing /etc/hosts
* List of [Vagrant Plugins](https://github.com/mitchellh/vagrant/wiki/Available-Vagrant-Plugins)
* Docs on [writing Vagrant plugins](http://docs.vagrantup.com/v1/docs/extending/)


---
## More Vagrant info

* [@mitchellh/vagrant](https://github.com/mitchellh/vagrant)
* [Vagrant Docs](http://docs.vagrantup.com/v1/docs)
* IRC: [#vagrant on freenode](http://irc.lc/freenode/vagrant)


---
## Chef-server

* [Open-source tool](http://community.opscode.com/) to manage the state of all your servers that are provisioned by chef-solo.
* developed by Opscode, who offer a [hosted version](http://www.opscode.com/hosted-chef/).


---
## Chef links

* http://wiki.opscode.com/display/chef/Home
* http://docs.opscode.com/
* [#ChefConf 2013](http://chefconf.opscode.com/) April 24-26 SFO
* [jtimperman's blog](http://jtimberman.housepub.org/) with advanced chef tips
* IRC: [#chef on freenode](http://irc.lc/freenode/chef)


---
## My vagrant/chef contribs

---
### Drupal and Redmine

* [@dergachev/vagrant_drupal](https://github.com/dergachev/vagrant_drupal) - work in progress
* [@dergachev/chef-redmine](https://github.com/dergachev/chef_redmine) cookbook
* [@dergachev/vagrant_redmine](https://github.com/dergachev/vagrant_redmine) tutorial; see extensive [NOTES.md](https://github.com/dergachev/vagrant_redmine/blob/master/NOTES.md)

---
### User provisioning with agent forwarding

* https://gist.github.com/3866825#vagrant-tutorial
* LWRP tutorial: [@dergachev/chef\_extend\_lwrp]( https://github.com/dergachev/chef_extend_lwrp)
* [@dergachev/chef\_root\_ssh_agent](https://github.com/dergachev/chef_root_ssh_agent) 
* [@dergachev/chef\_users](https://github.com/dergachev/chef_users)
* [@dergachev/chef\_homesick\_agent](https://github.com/dergachev/chef_homesick_agent), extends [@fnichol/chef-homesick](https://github.com/fnichol/chef-homesick)