# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'shell'

# The private network IP of the VM. You will use this IP to connect to OpenShift.
BASE_PUBLIC_ADDRESS = ENV['BASE_PUBLIC_ADDRESS'] || "10.50.1"
INFRA_ADDRESS = ENV['INFRA_ADDRESS'] || "#{BASE_PUBLIC_ADDRESS}.3"
BASE_HOSTNAME = ENV['BASE_HOSTNAME'] || "devops.bashburn.com"
OSE_HOSTNAME = ENV['OSE_HOSTNAME'] || "openshift.#{BASE_HOSTNAME}"
NUM_NODES = (ENV['NUM_NODES'] || 2).to_i

# Number of virtualized CPUs
VM_CPU = ENV['VM_CPU'] || 2

# Amount of available RAM
VM_MEMORY = ENV['VM_MEMORY'] || 2048

# Validate required plugins
REQUIRED_PLUGINS = %w(vagrant-service-manager vagrant-registration vagrant-hostmanager)
errors = []

def message(name)
  "#{name} plugin is not installed, run `vagrant plugin install #{name}` to install it."
end
# Validate and collect error message if plugin is not installed
REQUIRED_PLUGINS.each { |plugin| errors << message(plugin) unless Vagrant.has_plugin?(plugin) }
unless errors.empty?
  msg = errors.size > 1 ? "Errors: \n* #{errors.join("\n* ")}" : "Error: #{errors.first}"
  fail Vagrant::Errors::VagrantError.new, msg
end

cwd = Shell.new.cwd

Vagrant.configure(2) do |config|
  config.vbguest.auto_update = true
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true
  config.hostmanager.manage_host = true

  config.vm.provider "virtualbox" do |v, override|
    v.memory = VM_MEMORY
    v.cpus   = VM_CPU
    v.customize ["modifyvm", :id, "--ioapic", "on"]
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  config.vm.provider "libvirt" do |v, override|
    v.memory = VM_MEMORY
    v.cpus   = VM_CPU
    v.driver = "kvm"
  end

  if ENV.has_key?('RHN_USERNAME') && ENV.has_key?('RHN_PASSWORD') && ENV.has_key?('RHN_OSE_POOL')
    config.registration.username = ENV['RHN_USERNAME']
    config.registration.password = ENV['RHN_PASSWORD']
    config.registration.pools = [ ENV['RHN_OSE_POOL'] ]
  end
  config.vm.box = "rhel-7.2"

  config.vm.define "infra" do |infra|
    infra.vm.hostname = "infra"
    infra.vm.network "private_network", ip: "#{INFRA_ADDRESS}"
    infra.vm.network "forwarded_port", host: 2376, guest: 2376
    infra.hostmanager.aliases = %W(infra.#{BASE_HOSTNAME} gogs.#{BASE_HOSTNAME} nexus.#{BASE_HOSTNAME})
    infra.vm.synced_folder "../nexus", "/nexus"
    infra.vm.synced_folder "../jenkins", "/jenkins"
  end
  NUM_NODES.times do |i|
    node_index = i + 1
    config.vm.define "node#{node_index}" do |node|
      node.vm.hostname = "node#{node_index}"
      node.hostmanager.aliases = %W(node#{node_index}.#{BASE_HOSTNAME})
      node.vm.network "private_network", ip: "#{BASE_PUBLIC_ADDRESS}.#{100 + node_index}"
    end
  end
  config.vm.define "openshift" do |ose|
    ose.vm.hostname = "openshift"
    ose.hostmanager.aliases = %W(#{OSE_HOSTNAME})
    ose.vm.network "private_network", ip: "#{BASE_PUBLIC_ADDRESS}.200"
    ose.servicemanager.services = "docker"
    ose.vm.provision 'install', type: 'ansible' do |ansible|
      ansible.playbook = "config/devops-demo-playbook.yml"
      ansible.sudo = true
      ansible.limit = "all,localhost"
      ansible.verbose = false
      ansible.groups = {
        "infrastructure" => ["infra"],
        "nodes" => ["openshift"] + Array.new(NUM_NODES).map.with_index { |x, i| "node#{i + 1}" },
        "masters" => ["openshift"],
      }
      ansible.host_vars = {
        "openshift" => {
          "openshift_hostname" => "#{OSE_HOSTNAME}",
          "openshift_ip" => "#{BASE_PUBLIC_ADDRESS}.200",
          "openshift_public_ip" => "#{BASE_PUBLIC_ADDRESS}.200",
          "openshift_scheduleable" => true,
          "openshift_hosted_router_selector" => "region=infra",
          "openshift_registry_selector" => "region=infra",
          "openshift_node_labels" => "\"{'region': 'infra', 'zone': 'default'}\""
        }
      }
      NUM_NODES.times do |i|
        node_index = i + 1
        ansible.host_vars["node#{node_index}"] = {
          "openshift_hostname" => "node#{node_index}.#{BASE_HOSTNAME}",
          "openshift_ip" => "#{BASE_PUBLIC_ADDRESS}.#{100 + node_index}",
          "openshift_public_ip" => "#{BASE_PUBLIC_ADDRESS}.#{100 + node_index}",
          "openshift_hosted_router_selector" => "region=infra",
          "openshift_registry_selector" => "region=infra",
          "openshift_node_labels" => "\"{'region': 'primary', 'zone': 'default'}\""
        }
      end
      ansible.extra_vars = {
        "deployment_type" => 'openshift-enterprise',
        "base_hostname" => "#{BASE_HOSTNAME}",
        "base_address" => "#{BASE_PUBLIC_ADDRESS}",
        "infra_address" => "#{INFRA_ADDRESS}",
        "master_address" => "#{BASE_PUBLIC_ADDRESS}.200",
        "openshift_ansible_location" => "#{cwd}/external/openshift-ansible",
        "containerized" => false,
        "openshift_master_cluster_hostname" => "openshift.#{BASE_HOSTNAME}",
        "openshift_master_cluster_public_hostname" => "openshift.#{BASE_HOSTNAME}",
        "openshift_master_default_subdomain" => "#{BASE_HOSTNAME}",
        "openshift_master_identity_providers" => "[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true'," +
          "'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]"
      }
    end
    ose.vm.provision 'post-install', type: 'ansible' do |ansible|
      ansible.playbook = "config/devops-post-install.yml"
      ansible.sudo = true
      ansible.limit = "all"
    end
  end
end
