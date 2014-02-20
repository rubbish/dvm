# -*- mode: ruby -*-
# vi: set ft=ruby :
#
#  dvm - Effortless Docker-in-a-box for unsupported Docker platforms
#  For more details, please visit http://fnichol.github.io/dvm
#

ip      = ENV.fetch("DOCKER_IP", "192.168.42.43")
port    = ENV.fetch("DOCKER_PORT", "4243")
memory  = ENV.fetch("DOCKER_MEMORY", "512")
cpus    = ENV.fetch("DOCKER_CPUS", "1")
args    = ENV.fetch("DOCKER_ARGS", "")

if args.empty? && port != "4243"
  args = "-H unix:// -H tcp://0.0.0.0:#{port}"
end

module VagrantPlugins
  module GuestDvmTcl
    module Cap ; end

    class Plugin < Vagrant.plugin("2")
      name "Core Linux guest"
      description "Core Linux guest support"

      guest("tcl", "linux") do
        class ::VagrantPlugins::GuestDvmTcl::Guest < Vagrant.plugin("2", :guest)
          def detect?(machine)
            machine.communicate.test("cat /etc/issue | grep 'Core Linux'")
          end
        end
        Guest
      end

      guest_capability("tcl", "halt") do
        class ::VagrantPlugins::GuestDvmTcl::Cap::Halt
          def self.halt(machine)
            machine.communicate.sudo("poweroff")
          rescue IOError
            # Do nothing, because it probably means the machine shut down
            # and SSH connection was lost.
          end
        end
        Cap::Halt
      end

      guest_capability("tcl", "configure_networks") do
        class ::VagrantPlugins::GuestDvmTcl::Cap::ConfigureNetworks
          def self.configure_networks(machine, networks)
            require 'ipaddr'
            machine.communicate.tap do |comm|
              networks.each do |n|
                ifc = "/sbin/ifconfig eth#{n[:interface]}"
                broadcast = (IPAddr.new(n[:ip]) | (~ IPAddr.new(n[:netmask]))).to_s
                comm.sudo("#{ifc} down")
                comm.sudo("#{ifc} #{n[:ip]} netmask #{n[:netmask]} broadcast #{broadcast}")
                comm.sudo("#{ifc} up")
              end
            end
          end
        end
        Cap::ConfigureNetworks
      end
    end
  end
end

Vagrant.configure("2") do |config|
  config.vm.box = "boot2docker-0.5.4-1"
  config.vm.network "private_network", :ip => ip

  config.vm.provider :virtualbox do |v, override|
    override.vm.box_url = "https://github.com/mitchellh/boot2docker-vagrant-box/releases/download/v0.5.4-1/boot2docker_virtualbox.box"
    v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    v.customize ["modifyvm", :id, "--memory", Integer(memory)]
    v.customize ["modifyvm", :id, "--cpus", Integer(cpus)]
  end

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "https://github.com/mitchellh/boot2docker-vagrant-box/releases/download/v0.5.4-1/boot2docker_vmware.box"
      v.vmx["memsize"] = Integer(memory)
      v.vmx["numvcpus"] = Integer(cpus)
    end
  end

  config.vm.provision :shell, :inline => <<-PREPARE
    INITD=/usr/local/etc/init.d/docker
    if [ -n '#{args}' ] && grep -q 'docker -d .* $EXPOSE_ALL' $INITD >/dev/null; then
      echo "---> Configuring docker with args '#{args}' and restarting"
      sudo sed -i -e 's|docker -d .* $EXPOSE_ALL|docker -d #{args}|' $INITD
      sudo $INITD restart
    fi
    if ! grep -q '8\.8\.8\.8' /etc/resolv.conf >/dev/null; then
      echo "nameserver 8.8.8.8" >> /etc/resolv.conf
    fi
  PREPARE
  config.vm.define :dvm
end
