---
title: "DevOps 101: Chef Cookbook with Testing"
date: 2017-01-28T19:21:05-07:00
draft: false
comments: false
summary: A quick and dirty example of creating a chef cookbook with rspec tests.
tags:
  - devops
  - sre
categories:
  - deep-dive
  - software
toc: true
---
## Introduction


### Yet another chef tutorial

You can of course skip this step if you are familiar with Chef to a great enough degree, but in the spirit of making this a self-contained walkthrough, we will start from the ground up.

#### Getting started

##### Installing ChefDK

It is widely considered best practice to use [ChefDK](https://downloads.chef.io/chefdk), and this walkthrough uses ChefDK exclusively.

* On Mac, install [HomeBrew](http://brew.sh/) and run `brew cask install chefdk`
* On Linux, [download a package](https://downloads.chef.io/chefdk) corresponding to your distro and install accordingly.
* On Windows: Run Linux as a Virtual Machine, get Docker for Windows, or _try_ using Bash on Windows, but this walkthrough doesn't support Windows-based development.

##### Initiating your cookbook

Let's start with the following command:

```bash
chef generate cookbook c9_ide_chef
```

This command generates a template for a cookbook. Note that we use and underline in the name instead of a hyphen as hyphens can be problematic for chef in some circumstances. We now have a folder with the following content:

```bash
├── .gitignore
├── .kitchen.yml
├── Berksfile
├── README.md
├── chefignore
├── metadata.rb
├── recipes
│   └── default.rb
├── spec
│   ├── spec_helper.rb
│   └── unit
│       └── recipes
│           └── default_spec.rb
└── test
    └── smoke
        └── default
            └── default_test.rb
```
It also  creates a `.delivery` and a `.git` folder whose contents are omitted here for clarity.

The second command creates an attribute folder which we can populate with default attributes for our cookbook, and the third line creates a default template for a "message of the day" to be displayed when first logging in.

The `.git` directory is useful and we will use it as the basis for a new repo in github repo.

##### Pushing to GitHub

Now that we have the skeleton for our cookbook, we should start pushing to GitHub for version-control. On GitHub, create a public repo called c9_ide. Then run the following commands:

```bash
git add .
git commit -m "First commit"
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/c9_ide_chef.git
git push -u origin master
```

Now the stage is set and we are ready to make the cookbook actually do something.

#### Create the core configuration

##### Creating a story for the cookbook

So far we've seen how to set up the cookbook, but we don't quite know what we are going to do with it. Now is a good time to set up some goals for our cookbook:

* It should update all packages before doing anything else
* It should create a message of the day that gives basic information about the server to a user logging in
* It should install basic security packages and other utilities
* It should install the packages needed to run Cloud 9 IDE
* It should pull and install Cloud 9 IDE

###### Updating all packages

Just for fun, we are going to make our cookbook work for both Debian and RHEL variations of Linux. Under the recipes folder, let's create a recipe file called `update.rb` and give it contents:

```ruby
# Update system to start with
case node['platform']


when 'debian', 'ubuntu'

    execute 'AptUpdateUpgrade' do
        command "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get upgrade -y"
    end

when 'redhat', 'centos', 'fedora'

     # Update all packages
    execute 'UpdateYum' do
        command "yum -y update"
    end

    # Add Fedora's Extra Packages for Enterprise Linux
    execute 'GetEPELRepo' do
        command "rpm -iUvh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm"
        not_if "rpm -qa | grep -qx 'epel-release-7-9.noarch'"
    end

end
```

in the `default.rb` recipe, add the line:

```ruby
include_recipe 'c9_ide::update'
```

That's it for now! Before we go any further, let's test this. Chef has automatically added some basis for testing to your repo. Let's start with kitchen. This assumes you have [VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/) installed (if you don't, install them). At the base directory, run:
```bash
kitchen create
kitchen converge
```
The first command creates the virtual machines for us to work with, and teh second command cooks Chef inside of them. It is a very convenient way to do a full test of your code before pushing to github. The converge command may take quite a while as we are updating all packages on the systems and doing it for two VMs, and Ubuntu one and a CentOS one. If all goes well, the final output looks like:

```bash
Chef Client finished, 1/1 resources updated in 02 minutes 21 seconds
```

###### Creating a C9 User

It is rarely a good idea to run a program as root. As such, our next step is to create a user for running the Cloud 9 IDE. We start by creating a data bag. First, install knife-solo and [knife-solo data bag](https://github.com/thbishop/knife-solo_data_bag):

```bash
chef gem install knife-solo
chef gem install knife-solo_data_bag
```

This is not entirely necessary, but that tool can be useful in other ways for making chef solo cookbooks. Then we can generate a databag and databag item:

```bash
mkdir data_bags
export EDITOR="vi"
knife solo data bag create users c9ide
```

In the editor that pops up, paste the following content:

```json
{
  "id": "c9ide",
  "comment": "c9ide User",
  "ssh_keys": [
    ""
  ],
  "home": "/home/c9ide",
  "ssh_keygen": false
}
```

Next, back to `recipe/default.rb`, we can invoke the external chef recipe "users" to have this user installed on our system:

```ruby
include_recipe 'user::data_bag'
```

To create an attributes file:

```bash
chef generate attribute default
```

and in `attributes/default.rb` add the line:

```ruby
default['users'] = 'c9ide'
```

This of course shows the fantastic power and utility of Chef since we can call upon a wide range of libraries already created by the chef community. Now we need to tell Chef that we are using an external library, so in our `metadata.rb` file at the top level we add the line at the end:

```ruby
depends          'user'
```
At the same level there is a Berksfile with the content:

```ruby
source 'https://supermarket.chef.io'

metadata
```
which tells chef to check this `metadata.rb` file for dependencies (such as the user dependency we just added) that we need to pull and make available during cooking. We'll show how this is done manually a bit later, but for now `kitchen converge` takes care of that.

Now run `kitchen converge` one more time, and you should see that the user is created.

Let's create a quick smoke test item, add:


```ruby
describe user('c9ide') do
    it { should exist }
end
```

to `test/default/default_test.rb` and run

```bash
kitchen verify
```

if all went well you will see:

```bash
 User c9ide
     ✔  should exist
```

Now is a good time to push to github:

```bash
git add .
git commit -m "Adds update and users"
git push origin master
```


###### Installing MOTD

First, let's generate a blank template file:

```bash
chef generate template motd
```

fill with content:

```bash
<%= node['motd-attributes']['message'] %>
The hostname of this node is <%= node['hostname'] %>
The IP address of this node is <%= node['ipaddress'] %>
```

and create a recipe file `recipes/motd.rb` with contents:

```bash
template '/etc/motd' do
      source 'motd.erb'
      mode '0644'
end
```

and finally add this line to the `attributes/default.rb` file:

```ruby
default['motd-attributes']['message'] = "A Cloud 9 IDE server"
```

We add the following tests at `test/smoke/default/motd_test.rb`:

```ruby
describe directory('/etc/motd') do  # describe this directory
	its(:content) { should match /A Cloud 9 IDE server/ }
end
```
We run `kitchen converge` and `kitchen verify` again to make sure the recipe is fine, and then push to github as before.

###### Installing Packages

We are getting close to installing the Cloud9 IDE. Before doing so, we need to install some packages. Some of these packages are for security purposes, or convenience in administration, while others are needed for the Cloud 9 IDE system.

Create a file `recipe/packages.rb` with content:

```ruby
case node['platform']
when 'debian', 'ubuntu'

    req_packages = ['fail2ban',
                    'gcc',
                    'g++',
                    'git',
                    'htop',
                    'make',
                    'nodejs',
                    'nmap',
                    'npm',
                    'sysstat',
                    'unattended-upgrades',
                    'apt-listchanges']

    req_packages.each do |packs|
      Chef::Log.info("Installing: " + packs)
      package packs do
        action :install
      end
    end

    template '/etc/apt/apt.conf.d/50unattended-upgrades' do
        owner 'root'
        group 'root'
        mode '0644'
        source '50unattended-upgrades.erb'
    end

    template '/etc/apt/apt.conf.d/20auto-upgrades' do
        owner 'root'
        group 'root'
        mode '0644'
        source '20auto-upgrades.erb'
    end

when 'redhat', 'centos', 'fedora'

    req_packages = ['fail2ban',
                    'gcc-c++',
                    'git',
                    'glibc-static',
                    'htop',
                    'make',
                    'nodejs',
                    'nmap',
                    'npm',
                    'sysstat',
                    'yum-cron']

    req_packages.each do |packs|
      Chef::Log.info("Installing: " + packs)
      package packs do
        action :install
      end
    end

    template '/etc/yum/yum-cron.conf' do
        owner 'root'
        group 'root'
        mode '0644'
        source 'yum-cron.conf.erb'
    end

    execute 'systemctl enable yum-cron' do
    end

    execute 'systemctl start yum-cron' do
    end

end
```

Notice that we have sneaked in automated security updates as well. For that, we require the following templates:

`templates/50unattended-upgrades.erb`:

```bash
// Automatically upgrade packages from these (origin, archive) pairs
Unattended-Upgrade::Allowed-Origins {
  <% node['unattended-upgrades']['origins'].each do |origin| %>
  "<%= origin %>";
  <% end %>
};

// List of packages to not update
Unattended-Upgrade::Package-Blacklist {
  "apache2";
  "nginx";
//  "libc6-dev";
//  "libc6-i686";
};

// Send email to this address for problems or packages upgrades
// If empty or unset then no email is sent, make sure that you
// have a working mail setup on your system. The package 'mailx'
// must be installed or anything that provides /usr/bin/mail.
<%= node['unattended-upgrades']['send_email'] ? "" : "// " %>Unattended-Upgrade::Mail "<%= node['unattended-upgrades']['email_address'] %>";

// Do automatic removal of new unused dependencies after the upgrade
// (equivalent to apt-get autoremove)
Unattended-Upgrade::Remove-Unused-Dependencies "<%= node['unattended-upgrades']['auto_remove'] ? "true" : "false" %>";

// Automatically reboot *WITHOUT CONFIRMATION* if a
// the file /var/run/reboot-required is found after the upgrade
Unattended-Upgrade::Automatic-Reboot "<%= node['unattended-upgrades']['auto_reboot'] ? "true" : "false" %>";
```

`templates/20auto-upgrades.erb`:

```bash
APT::Periodic::Update-Package-Lists "<%=node['unattended-upgrades']['update_package_lists_interval']%>";
APT::Periodic::Unattended-Upgrade "<%=node['unattended-upgrades']['upgrade_interval']%>";
```

and

`templates/yum-cron.conf.erb`:

```bash
update_cmd = security
apply_updates = yes
```

```bash
default['unattended-upgrades']['update_package_lists_interval'] = "1"
default['unattended-upgrades']['upgrade_interval'] = "1"
default['unattended-upgrades']['origins'] = ['${distro_id} ${distro_codename}-security']
default['unattended-upgrades']['send_email'] = false
default['unattended-upgrades']['email_address'] = "test@example.com"
default['unattended-upgrades']['auto_remove'] = false
default['unattended-upgrades']['auto_reboot'] = false
```

and of course, our inspec test in `test/smoke/default/packages_test.rb`:

```ruby
if os[:family] == 'debian'
  	req_packages = ['fail2ban',
                    'gcc',
                    'g++',
                    'git',
                    'htop',
                    'make',
                    'nodejs',
                    'nmap',
                    'npm',
                    'sysstat',
                    'unattended-upgrades',
                    'apt-listchanges']

    req_packages.each do |packs|
      describe package(packs) do
      	it { should be_installed }
      end
    end

else

    req_packages = ['fail2ban',
                    'gcc-c++',
                    'git',
                    'glibc-static',
                    'htop',
                    'make',
                    'nodejs',
                    'nmap',
                    'npm',
                    'sysstat',
                    'yum-cron']

    req_packages.each do |packs|
      describe package(packs) do
      	it { should be_installed }
      end
    end
end
```

###### Installing Cloud 9 IDE

Chef has a built-in git resource that we can use to pull the Cloud 9 IDE code.

The test for this section will be rather brief:

```ruby
describe directory('/home/c9ide/core/README.md') do  # describe this directory
	its(:content) { should match /^Cloud9/ }
end
```

and the reason is that we can really verify whether or not this section succeeded in the next section on running via supervisor.

The code for this section, in `/recipes/c9ide.rb`, is:

```ruby
git "/home/c9ide/core" do
   repository 'https://github.com/c9/core.git'
   reference 'master'
   action :sync
end

case node['platform']
when 'debian', 'ubuntu'
	bash 'create nodejs link' do
		code <<-EOH
		ln -s /usr/bin/nodejs /usr/bin/node
		EOH
		not_if { ::File.exist?('/usr/bin/node') }
	end
end

bash 'install_cloud9ide' do
   cwd '/home/c9ide/core/'
   user 'root'
   code <<-EOH
     scripts/install-sdk.sh
     EOH
   environment 'PREFIX' => '/usr/local'
end
```

###### Installing Supervisor

Finally, we want to leave our server running Cloud 9 IDE after provisioning it. Supervisor is a fantastic way to do so, though of course not the only way and perhaps not even the best way, but it is a convenient and production-ready solution (though this development kit version of Cloud 9 IDE is not production material and is meant for development purposes).

In `recipes/supervisor.rb`:

```ruby
package 'supervisor' do
    action :install
end

case node['platform']
when 'debian', 'ubuntu'

	cookbook_file '/etc/supervisor/conf.d/cloud9ide.conf' do
	  source 'c9ide.conf'
	  owner 'root'
	  group 'root'
	  mode '0755'
	  action :create
	end

	execute 'start supervisor' do
	  command 'service supervisor start'
	end

	execute 'reload supervisor' do
	  command 'supervisorctl update'
	end

when 'redhat', 'centos', 'fedora'

	cookbook_file '/etc/supervisord.d/cloud9ide.ini' do
	  source 'c9ide.conf'
	  owner 'root'
	  group 'root'
	  mode '0755'
	  action :create
	end

	execute 'start supervisor' do
	  command 'service supervisord start'
	end

	execute 'reload supervisor' do
	  command 'supervisorctl update'
	end
end
```

and finally add this to the `c9ide_test.rb` test file:


```ruby
describe port(9999) do
  it { should be_listening }
end
```

##### Putting it all together

The default recipe should now look like this:

```ruby
include_recipe 'c9_ide::update'
include_recipe 'user::data_bag'
include_recipe 'c9_ide::motd'
include_recipe 'c9_ide::packages'
include_recipe 'c9_ide::cloud9ide'
include_recipe 'c9_ide::supervisor'
```

we run a final `kitchen converge`; if all goes well, we follow up with a final `kitchen verify`:

```bash
-----> Starting Kitchen (v1.14.2)
-----> Setting up <default-ubuntu-1604>...
       Finished setting up <default-ubuntu-1604> (0m0.00s).
-----> Verifying <default-ubuntu-1604>...
       Loaded

Target:  ssh://vagrant@127.0.0.1:2222


  File /home/c9ide/core/README.md
     ✔  content should match /^Cloud9/
  Port 9999
     ✔  should be listening
  User c9ide
     ✔  should exist
  File /etc/motd
     ✔  content should match /A Cloud 9 IDE server/
  System Package
     ✔  fail2ban should be installed
  System Package
     ✔  gcc should be installed
  System Package
     ✔  g++ should be installed
  System Package
     ✔  git should be installed
  System Package
     ✔  htop should be installed
  System Package
     ✔  make should be installed
  System Package
     ✔  nodejs should be installed
  System Package
     ✔  nmap should be installed
  System Package
     ✔  npm should be installed
  System Package
     ✔  sysstat should be installed
  System Package
     ✔  unattended-upgrades should be installed
  System Package
     ✔  apt-listchanges should be installed

Test Summary: 16 successful, 0 failures, 0 skipped
       Finished verifying <default-ubuntu-1604> (0m1.28s).
-----> Setting up <default-centos-72>...
       Finished setting up <default-centos-72> (0m0.00s).
-----> Verifying <default-centos-72>...
       Loaded

Target:  ssh://vagrant@127.0.0.1:2200


  File /home/c9ide/core/README.md
     ✔  content should match /^Cloud9/
  Port 9999
     ✔  should be listening
  User c9ide
     ✔  should exist
  File /etc/motd
     ✔  content should match /A Cloud 9 IDE server/
  System Package
     ✔  fail2ban should be installed
  System Package
     ✔  gcc-c++ should be installed
  System Package
     ✔  git should be installed
  System Package
     ✔  glibc-static should be installed
  System Package
     ✔  htop should be installed
  System Package
     ✔  make should be installed
  System Package
     ✔  nodejs should be installed
  System Package
     ✔  nmap should be installed
  System Package
     ✔  npm should be installed
  System Package
     ✔  sysstat should be installed
  System Package
     ✔  yum-cron should be installed

Test Summary: 15 successful, 0 failures, 0 skipped
       Finished verifying <default-centos-72> (0m13.80s).
-----> Kitchen is finished. (0m16.47s)
```

Congratulations, you have a basic working chef recipe for installing Cloud 9 IDE, tested and provisioned on both major families of Linux.

### Source Code

You can find the source code for this recipe on my github account [here](https://github.com/3implieschaos/c9_ide_chef/tree/PartI).

### Future Steps

We took some shortcuts and avoided some best practices for convenience here. Ideally we would like some unit tests and to pull in more community recipes. For example, the supervisor chef cookbook could be used here instead of dealing with that in a platform case.

In the forthcoming Part II, we will use Puppet, a widely used alternative to Chef, to provision a server that runs the Cloud 9 IDE SDK as we have here with Chef.
