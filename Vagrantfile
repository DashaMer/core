#!/usr/bin/env ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

puts "
==================================================================
To change provisioning options please use source.[provider] files.
For more information refere to https://github.com/pintostack/core
"
system("ls source.*")
puts "Usage:
vagrant up --provider=[virtualbox|aws|digital_ocean]

More information on vagrant it self you can find
here http://docs.vagrantup.com/v2/cli/index.html
==================================================================
"

if ARGV[1] and \
   (ARGV[1].split('=')[0] == "--provider" or ARGV[2])
  provider = (ARGV[1].split('=')[1] || ARGV[2])
else
  provider = (ENV['VAGRANT_DEFAULT_PROVIDER'] || :virtualbox).to_sym
end

if  provider == "aws" 
  vpc_if="eth0"
else
  vpc_if="eth1"
end
puts "Detected #{provider}..."
puts "Setting VPC interface #{vpc_if}..."

SLAVES = %x( bash -c "source source.global && echo \\$SLAVES" ).strip
MASTERS = %x( bash -c "source source.global && echo \\$MASTERS" ).strip

#Vagrant.require_plugin 'vagrant-aws'
#Vagrant.require_plugin 'vagrant-digital_ocean'
#Vagrant.require_plugin 'vagrant-cachier'

ALL_HOSTS=[]
(1..MASTERS.to_i).each do |i|
   m_host = "master-#{i}"
   ALL_HOSTS.push("#{m_host}")
end
(1..SLAVES.to_i).each do |i|
    s_host = "slave-#{i}"
    ALL_HOSTS.push("#{s_host}")
end
ALL_HOSTS.push("docker-registry")

Vagrant.configure(2) do |config|
  config.vm.network "private_network", type: :dhcp
  config.vm.hostname = 'default-host-delete-me'
  config.vm.box = "ubuntu/trusty64"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provider :digital_ocean do |digital_ocean, override|
          digital_ocean.token = %x( bash -c "source source.digital_ocean && echo \\$DO_TOKEN").strip
          digital_ocean.image = %x( bash -c "source source.digital_ocean && echo \\$DO_IMAGE").strip
          digital_ocean.region = %x( bash -c "source source.digital_ocean && echo \\$DO_REGION").strip
          digital_ocean.size = %x( bash -c "source source.digital_ocean && echo \\$DO_SIZE").strip
          digital_ocean.private_networking = true
          override.ssh.private_key_path = %x( bash -c "source source.digital_ocean && echo \\$SSH_KEY_FILE").strip
          override.vm.box = 'digital_ocean'
          override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"
  end
  config.vm.provider :aws do |aws, override|
          aws.access_key_id = %x( bash -c "source source.aws && echo \\$AWS_KEY_ID").strip
          aws.secret_access_key = %x( bash -c "source source.aws && echo \\$AWS_ACCESS_KEY").strip
          aws.keypair_name = %x( bash -c "source source.aws && echo \\$AWS_KEYPAIR_NAME").strip
          aws.ami = %x( bash -c "source source.aws && echo \\$AWS_AMI").strip
          aws.instance_type = %x( bash -c "source source.aws && echo \\$AWS_INSTANCE_TYPE").strip
          aws.region = %x( bash -c "source source.aws && echo \\$AWS_REGION").strip
          aws.security_groups = %x( bash -c "source source.aws && echo \\$AWS_SECURITY_GROUPS").strip.split(",")
          aws.block_device_mapping = [{ 'DeviceName' => '/dev/sda1', 'Ebs.VolumeSize' => %x( bash -c "source source.aws && echo \\$AWS_ROOT_PARTITION_SIZE").strip }]
          override.ssh.username = %x( bash -c "source source.aws && echo \\$AWS_SSH_USERNAME").strip
          override.vm.box = 'aws'
          override.vm.box_url ="https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
          override.ssh.private_key_path = %x( bash -c "source source.aws && echo \\$SSH_KEY_FILE").strip
  end
  config.vm.provider :virtualbox do |virtualbox|
          virtualbox.gui = false
          virtualbox.memory = %x( bash -c "source source.virtualbox && echo \\$VBOX_SIZE").strip
          virtualbox.cpus = 2
  end

  ALL_HOSTS.each do |i|
    name = "#{i}"
    config.vm.define name do |instance|
	    instance.vm.hostname = name # Name in web console DO
	    instance.vm.provider :aws do |aws, override|
        aws.tags={ 'Name' => name }
      end
      instance.vm.provider :virtualbox do |virtualbox|
          virtualbox.name = name
      end
      if name == "docker-registry" # Run ansible after last host provision
        instance.vm.provision :ansible do |ansible|
          ansible.playbook = "provisioning/world-playbook.yml"
          ansible.limit = 'all'
          ansible.force_remote_user = true
          ansible.host_vars={}
          ALL_HOSTS.each do |each_host|
            ansible.host_vars["#{each_host}"] = {"vpc_if" => "#{vpc_if}"}
          end
          ansible.groups = {
              "masters" => ["master-[1:#{MASTERS.to_i}]"],
              "slaves" => ["slave-[1:#{SLAVES.to_i}]"],
              "dockers" => ["docker-registry"],
              "all-hosts" => ["master-[1:#{MASTERS.to_i}]","slave-[1:#{SLAVES.to_i}]","docker-registry"]
          }
        end
      end
    end
  end
#END
end


ANSIBLE_INVENTORY_FILE = %x( bash -c "source source.global && echo \\$ANSIBLE_INVENTORY_FILE" ).strip

puts "
==================================================================
After vagrant up --provider=[provider] finished you can rerun 
ansible script on your new cluster by executing:

ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory provisioning/world-playbook.yml

"
system("open_webui.sh")

