# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :"hw09-otus" => {
        :box_name => "zradeg/hw09-otus",
        :ip_addr => '192.168.11.102'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "256"]
          end
          
          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
			dnf install -y vim
          SHELL

      end
      config.vbguest.auto_update = false
  end
end
