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
