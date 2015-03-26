# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'
require 'json'

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "user-data")

#xdevel - personal config on json devops.json
XDEV_CONFIG = JSON.parse(File.read(File.join(File.dirname(__FILE__), 'devops.json')))
#xdevel - aditional ports, if necessary by others vm's
DOCKER_PORTS = XDEV_CONFIG["aditional_ports"]

#xdevel - volumes 
DOCKER_VOLUMES = XDEV_CONFIG["folders"]


#xdevel - first key for default (main vm will use in first time)
DEFAULT_KEY_PATH = "~/.vagrant.d/insecure_private_key"



# Defaults for config options defined in CONFIG
# xdevel - adjust the config from devops.config
$num_instances = XDEV_CONFIG["num_instances"]
$update_channel = XDEV_CONFIG["update_channel"]
$enable_serial_logging = XDEV_CONFIG["enable_serial_logging"]
$share_home = XDEV_CONFIG["share_home"]
$vm_gui = XDEV_CONFIG["virtualbox"]["vb_gui"]
$vm_memory = XDEV_CONFIG["virtualbox"]["vb_memory"]
$vm_cpus = XDEV_CONFIG["virtualbox"]["vb_cpus"]
$expose_docker_tcp = XDEV_CONFIG["expose_docker_tcp"]
$share_home = XDEV_CONFIG["share_home"]

#xdevel - version of coreos - minimal ou equal
$coreos_version = XDEV_CONFIG["coreos-version"]

# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end


# Use old vb_xxx config variables when set
def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

Vagrant.configure("2") do |config|

  # b2d doesn't persist filesystem between reboots (https://github.com/mitchellh/vagrant/blob/master/plugins/providers/docker/hostmachine/Vagrantfile)
  #if config.ssh.respond_to?(:insert_key)
  #  config.ssh.insert_key = false
  #end

  #xdevel - don't insert insecure key
  config.ssh.username = "core"
  config.ssh.insert_key = false
  
  #xdevel - insert your personal key on vm
  config.ssh.private_key_path = [DEFAULT_KEY_PATH,XDEV_CONFIG["private_key_path"]]

  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 522.5.0"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel


  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant_vmware_fusion.json" % $update_channel
    end
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "core-%02d" % i do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        ["vmware_fusion", "vmware_workstation"].each do |vmware|
          config.vm.provider vmware do |v, override|
            v.vmx["serial0.present"] = "TRUE"
            v.vmx["serial0.fileType"] = "file"
            v.vmx["serial0.fileName"] = serialFile
            v.vmx["serial0.tryNoRxLoss"] = "FALSE"
          end
        end

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      DOCKER_PORTS.each do |port|
        config.vm.network "forwarded_port", guest: port, host: port, auto_correct: false
      end

      

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        config.vm.provider vmware do |v|
          v.gui = vm_gui
          v.vmx['memsize'] = vm_memory
          v.vmx['numvcpus'] = vm_cpus
        end
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = vm_gui
        vb.memory = vm_memory
        vb.cpus = vm_cpus
      end

      ip = "172.17.8.#{i+100}"
      config.vm.network :private_network, ip: ip
      #bug conform link (https://github.com/mitchellh/vagrant/issues/4011)
      config.nfs.functional = true


      DOCKER_VOLUMES.each do |folder|
        #config.vm.synced_folder folder["src"], folder["dest"], owner: folder["uid"], group: folder["gid"] 
        #config.vm.synced_folder folder["src"], folder["dest"], owner: folder["uid"], group: folder["gid"] 


        config.vm.synced_folder folder["src"], folder["dest"], :nfs => true, :mount_options => ['rw,nolock,vers=3,udp']
        config.vm.synced_folder folder["src"], folder["dest"], :nfs => true, :mount_options => ['rw,nolock,vers=3,udp']

      end

      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']

      #xdevel - data files from enviroments
      #config.vm.synced_folder "./data", "/home/core/data", id: "core", :nfs => true, :mount_options => ['rw,nolock,vers=3,udp']
      #config.vm.synced_folder "./data", "/home/core/data", id: "core"


      if $share_home
        config.vm.synced_folder ENV['HOME'], ENV['HOME'], id: "home", :nfs => true, :mount_options => ['rw,nolock,vers=3,udp']
      end



      if File.exist?(CLOUD_CONFIG_PATH)
        config.vm.provision :file, :source => "#{CLOUD_CONFIG_PATH}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
        #xdevel - adding  docker client and  run daemon
        
        #xdevel - restart the services
        #config.vm.provision :shell, :inline => "systemctl start docker-local.service", :privileged => true
        #config.vm.provision :shell, :inline => "systemctl start etcd.service", :privileged => true
        #config.vm.provision :shell, :inline => "systemctl start fleet.service", :privileged => true


        #xdevel - owner problem in folders - workarround this :( (descobri a besteira que cometi)
        #DOCKER_VOLUMES.each do |folder|
        #  $cmd = "chown -R " + folder["uid"] + ":" + folder["gid"]  + " " + folder["dest"]
        #  config.vm.provision :shell, :inline => $cmd , :privileged => true
        #end

      end

    end
  end
end
