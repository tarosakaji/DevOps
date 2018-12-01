DevOps practice - Create box with Packer and running with Vagrant
=================================================================

These instructions assume familiarity with Git and GitHub. If you are not comfortable with those tools, please complete Udacity's [How to Use Git and GitHub](https://www.udacity.com/course/how-to-use-git-and-github--ud775) course before proceeding. 

After installing the required tools, you will need to ensure that your computer can find the executables to run them. For this, you might need to modify the PATH environment variable. A good overview is at [superuser.com](https://superuser.com/questions/284342/what-are-path-and-other-environment-variables-and-how-can-i-set-or-use-them). You may need to search the web for instructions on how to set the PATH variable for your specific operating system and version. 

## Setting up your local machine

* Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* Install [Vagrant](https://www.vagrantup.com/downloads.html)
* Install [Packer](https://www.packer.io/downloads.html)
* Fork this repo to your own account
* Clone the forked repo to your local machine using this command: `git clone http://github.com/<account-name>/devops-intro-project devops`, replacing `<account-name>` with your GitHub username.

## Part I: Building a box with Packer

From the packer-templates directory on your local machine:

* Run `packer build -only=virtualbox-iso application-server.json`. You may see various timeouts and errors, as shown below. If you do, retry the command until the ISO download succeeds:

```
read: operation timed out
==> virtualbox-iso: ISO download failed.
Build 'virtualbox-iso' errored: ISO download failed.

checksums didn't match expected
==> virtualbox-iso: ISO download failed.
Build 'virtualbox-iso' errored: ISO download failed.

==> Some builds didn't complete successfully and had errors:
--> virtualbox-iso: ISO download failed.
```

* Run `cd virtualbox`
* Run `vagrant box add ubuntu-14.04.4-server-amd64-appserver_virtualbox.box --name devops-appserver`
* Run `vagrant init` to initialize the vagrant and create the `vagrantfile` file if is not created automatically
Inside the `vagrantfile` should have some configuration info like:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. 

Vagrant.configure(2) do |config|

  config.vm.box = "devops-controlserver"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder.
  config.vm.synced_folder "archivos", "/home/vagrant/archivos", create: true

end
```
* Run `vagrant up`
* Run `vagrant ssh` to connect to the server


## Part II: Cloning, developing, and running the web application

* On your local machine go to the root directory of the cloned repository 
* Run `git clone https://github.com/chef/devops-kungfu.git devops-kungfu`
* Open http://localhost:8080 from your local machine to see the app running.
* In the VM, run `cd devops-kungfu`
* To install app specific node packages, run `sudo npm install`. You may see several errors; they can be ignored for now.
* Now you can run tests with the command `grunt -v`. The tests will run, then quit with an error. 

### Troubleshooting

If you encounter errors with Ubuntu version numbers not being available or checksum errors on Ubuntu,it means that this repository has not yet been updated for the latest Ubuntu version. Feel free to mention this in the [forum](https://discussions.udacity.com/c/nd012-p1-intro-to-devops/nd012-the-devops-environment). Meanwhile, you can fix this error for yourself by editing the contents of the `application-server.json` and `control-server.json` template files inside the `packer-templates` folder.

* Find the newest version number and checksum from the [Ubuntu website for this release](http://releases.ubuntu.com/trusty/)
* Edit `PACKER_BOX_NAME` and `iso_checksum` in the template files to match that version number and checksum.

### ALE NOTES: Note about Packer and Vagrant with OWS:
======================================================
* Is necessary to configure a custom security group opening the SSH port in order to being able to access the virtual machine from OWS.
* To deploy packer on OWS need to run like this: `packer build -var 'AWS_ACCESS_KEY_ID=XXXXX' -var 'AWS_SECRET_ACCESS_KEY=XXXXXXX' -only=amazon-ebs control-server.json`
* To use Vagrant on OWS need to:
1. install: `vagrant plugin install vagrant-aws`
2. configure your golden box with `vagrant box add aws ubuntu-14.04.5-server-amd64-appserver_aws.box`
3. run `vagrant init` to initialize the vagrant file and then configure like this:
```
# Require the AWS provider plugin
require 'vagrant-aws'

# Creating and configuring the AWS instance
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  # Use the AWS box
  config.vm.box = 'aws'

  # Specify configuration of AWS provider
  config.vm.provider 'aws' do |aws, override|
  config.vm.allowed_synced_folder_types = [:rsync]

  # Read AWS authentication information from environment variables
  aws.access_key_id = ENV['AWS_ACCESS_KEY_ID']
  aws.secret_access_key = ENV['AWS_SECRET_ACCESS_KEY']

  # Specify SSH keypair to use
  aws.keypair_name = 'ows-vagrant'

  # Specify region, AMI ID, Instance and security group
  aws.region = 'us-east-1'
  aws.ami = 'ami-10b68a78'
  aws.instance_type = 't1.micro'
  aws.security_groups = ['vagrant'] 

  
  # Specify username and private key path
  override.ssh.username = 'ubuntu'
  override.ssh.private_key_path = 'ows-vagrant.pem'
  end
 end

```
 4. Execute passing the environment variables like this `AWS_ACCESS_KEY_ID='xxxxxxx' AWS_SECRET_ACCESS_KEY='xxxx+xxxxx' vagrant up`.

 5. Then to connect through SSH you could use the `AWS_ACCESS_KEY_ID='xxxxxxx' AWS_SECRET_ACCESS_KEY='xxxx+xxxxx' vagrant ssh` command.

### ALE NOTES: Packer vs Docker
================================
1. Packer needs a .json file to create a golden image with the definition of 
 a. Variables
 b. Builders: Operating Systems
 c. Provisioners: Install and configure the software
 d. PostProcesors: take the previous result and create a new artifact (a build)

 Example: 
 Local Deployment ```packer build -only=virtualbox-iso application-server.json```
 OWS Deployment ```packer build -var 'AWS_ACCESS_KEY_ID=xxxx' -var 'AWS_SECRET_ACCESS_KEY=xxxx' -only=amazon-ebs control-server.json```

2. Docker needs a ```docker-compose.yml file``` in which you define the image container to be deployed. 
 a. Execute ```docker-compose up -d``` to create the Docker
 b. To verify the execution you could use ```docker ps```
 c. Also could verify processor/memory consumption with ```docker stats```

