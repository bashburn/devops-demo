#
# Rakefile
#
#   Build commands for managing the vagrant environment
#
require 'open4'

def exec_cmd(arg)
  Open4::popen4(arg) do |pid, stdin, stdout, stderr|
    stdout.each do |line|
      puts line
    end
  end
end

desc 'Shutdown all VMs and remove their files'
task :clean do
  exec_cmd 'vagrant destroy -f'
end

desc 'Destroy only the infrastructure VM'
task :clean_infra do
  exec_cmd 'vagrant destroy -f infra'
end

desc 'Destroy only the OpenShift VM'
task :clean_openshift do
  exec_cmd 'vagrant destroy -f openshift'
end

task :start do
  status = 0
  Open4::popen4('vagrant status') do |pid, stdin, stdout, stderr|
    stdout.each do |line|
      puts line
      status &= 1 if /^infra[ \t]running.*$/ =~ line
      status &= 2 if /^openshift[ \t]*running.*$/ =~ line
    end
  end
  exec_cmd 'vagrant up' if status == 0
  exec_cmd 'vagrant up openshift' if status == 1
  exec_cmd 'vagrant up infra' if status == 2
end

task :start_infra do
  exec_cmd 'vagrant up infra'
end

task :start_openshift do
  exec_cmd 'vagrant up openshift'
end

task :provision_infra do
  exec_cmd 'rm config/*.retry'
  exec_cmd 'vagrant provision infra'
end
