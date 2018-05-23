# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

current_dir         = File.dirname(File.expand_path(__FILE__))
config              = YAML.load_file("#{current_dir}/vagrant_config.yml")["config"]

BOX_IMAGE           = config["box"]
NETWORK             = config["network"].split(".").take(3).join(".")
MASTER_NODE_COUNT   = config["master"]["count"]
WORKER_NODE_COUNT   = config["worker"]["count"]
MASTER_NODE_MEMORY  = config["master"]["memory"]
MASTER_NODE_CPU     = config["master"]["cpu"]
WORKER_NODE_MEMORY  = config["worker"]["memory"]
WORKER_NODE_CPU     = config["worker"]["cpu"]
CLIENT_NODE_ENABLED = config["client"]["enabled"]
CLIENT_NODE_MEMORY  = config["client"]["memory"]
CLIENT_NODE_CPU     = config["client"]["cpu"]

Vagrant.configure("2") do |config|
  (1..MASTER_NODE_COUNT).each do |i|
    config.vm.define "master#{i}" do |subconfig|
      subconfig.vm.box      = BOX_IMAGE
      subconfig.vm.hostname = "master#{i}"
      
      subconfig.vm.network :private_network,
        ip: "#{NETWORK}.#{i + 10}"

      subconfig.vm.provider "virtualbox" do |vb|
        vb.memory = MASTER_NODE_MEMORY
        vb.cpus   = MASTER_NODE_CPU
      end
    end
  end

  (1..WORKER_NODE_COUNT).each do |i|
    config.vm.define "worker#{i}" do |subconfig|
      subconfig.vm.box      = BOX_IMAGE
      subconfig.vm.hostname = "worker#{i}"
      
      subconfig.vm.network :private_network,
        ip: "#{NETWORK}.#{i + 20}"

      subconfig.vm.provider "virtualbox" do |vb|
        vb.memory = WORKER_NODE_MEMORY
        vb.cpus   = WORKER_NODE_CPU
      end
    end
  end

  if CLIENT_NODE_ENABLED then
    config.vm.define "client" do |client|
      client.vm.box =      BOX_IMAGE
      client.vm.hostname = "client"

      client.vm.network :private_network,
        ip: "#{NETWORK}.9"
      
      client.vm.provider "virtualbox" do |vb|
        vb.memory = CLIENT_NODE_MEMORY
        vb.cpus   = CLIENT_NODE_CPU
      end
    end
  end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y avahi-daemon libnss-mdns
  SHELL
end
