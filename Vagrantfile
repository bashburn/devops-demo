# -*- mode: ruby -*-
# vi: set ft=ruby :

# The private network IP of the VM. You will use this IP to connect to OpenShift.
PUBLIC_ADDRESS="10.1.2.2"
INFRA_ADDRESS="10.1.2.3"

BASE_HOSTNAME="devops.bashburn.com"
HOSTNAME = "openshift.#{BASE_HOSTNAME}"

# Number of virtualized CPUs
VM_CPU = ENV['VM_CPU'] || 2

# Amount of available RAM
VM_MEMORY = ENV['VM_MEMORY'] || 3072

# Validate required plugins
REQUIRED_PLUGINS = %w(vagrant-service-manager vagrant-registration)
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

set_resolv_conf = <<-SHELL
      echo "
domain #{BASE_HOSTNAME}
nameserver #{PUBLIC_ADDRESS}
" > /etc/resolv.conf
SHELL

Vagrant.configure(2) do |config|
  config.vm.box = "bashburn/cdkv2"
  config.hostmanager.enabled = false
  config.vbguest.auto_update = false

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
  config.vm.define "infra.#{BASE_HOSTNAME}" do |infra|
    config.vm.box = "rhel-7.2"
    config.vm.hostname = "infra.#{BASE_HOSTNAME}"
    config.vm.network "private_network", ip: "#{INFRA_ADDRESS}"
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "config/infra-playbook.yml"
      ansible.groups = {
        "infra" => ["infra.devops.bashburn.com"],
        "infra:vars" => {"mysql_password" => "devops"}
      }
    end
    # config.vm.provision "shell", inline: set_resolv_conf
  end
  config.vm.define HOSTNAME do |ose|
    config.vm.hostname = HOSTNAME
    config.vm.network "private_network", ip: "#{PUBLIC_ADDRESS}"
    config.servicemanager.services = "docker"

    config.vm.provision "shell", inline: <<-SHELL
      systemctl enable openshift 2>&1
      systemctl start openshift

      echo "
#/etc/dnsmasq.conf
domain-needed
bogus-priv

address=/.#{BASE_HOSTNAME}/#{PUBLIC_ADDRESS}
server=8.8.8.8
" > /etc/dnsmasq.conf
      echo "
domain #{BASE_HOSTNAME}
nameserver #{PUBLIC_ADDRESS}
" > /etc/resolv.conf
    SHELL

    config.vm.provision "shell", inline: <<-SHELL
      echo
      echo "Successfully started and provisioned VM with #{VM_CPU} cores and #{VM_MEMORY} MB of memory."
      echo "To modify the number of cores and/or available memory set the environment variables"
      echo "VM_CPU respectively VM_MEMORY."
      echo
      echo "You can now access the OpenShift console on: https://#{PUBLIC_ADDRESS}:8443/console"
      echo
      echo "To use OpenShift CLI, run:"
      echo "$ vagrant ssh"
      echo "$ oc login #{PUBLIC_ADDRESS}:8443"
      echo
      echo "Configured users are (<username>/<password>):"
      echo "openshift-dev/devel"
      echo "admin/admin"
      echo
      echo "If you have the oc client library on your host, you can also login from your host."
      echo
    SHELL
  end

end
