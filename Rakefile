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

desc 'Start all VMs that have not been started'
task :start do
  exec_cmd 'vagrant up'
end

desc 'Run the ansible post-install playbook to get the environment ready'
task :post_install do
  exec_cmd 'vagrant provision openshift --provision-with post-install'
end

task :remove_efk do
  exec_cmd 'oc delete all -n logging --selector logging-infra=kibana'
  exec_cmd 'oc delete all -n logging --selector logging-infra=fluentd'
  exec_cmd 'oc delete all -n logging --selector logging-infra=elasticsearch'
  exec_cmd 'oc delete all -n logging --selector logging-infra=curator'
  exec_cmd 'oc delete all,sa,oauthclient -n logging --selector logging-infra=support'
  exec_cmd 'oc delete secret logging-fluentd logging-elasticsearch logging-es-proxy logging-kibana logging-kibana-proxy logging-kibana-ops-proxy'
end

task :remove_metrics do
  exec_cmd 'oc delete all --selector="metrics-infra"'
  exec_cmd 'oc delete sa --selector="metrics-infra"'
  exec_cmd 'oc delete templates --selector="metrics-infra"'
  exec_cmd 'oc delete secrets --selector="metrics-infra"'
  exec_cmd 'oc delete pvc --selector="metrics-infra"'
  exec_cmd 'oc delete sa metrics-deployer'
  exec_cmd 'oc delete secret metrics-deployer'
end
