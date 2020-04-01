# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :"hw09-otus" => {
        :box_name => "centos/8",
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
			groupadd admin
			usermod -aG admin root
			useradd test1 -s /bin/bash
			echo "test1:test1" | chpasswd
			useradd duty -s /bin/bash
			echo "duty:duty" | chpasswd
			usermod -aG admin duty
			sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
			systemctl restart sshd
			echo '#!/bin/bash
ID_USER=$(id $PAM_USER)
CHECKGROUP=$(echo ${ID_USER} | grep -o admin)
echo ${CHECKGROUP}
if [[ ${CHECKGROUP} != admin ]]; then
 if [ $(date +%a) = "Sat" ] || [ $(date +%a) = Sun ]; then
 exit 1
 else
 exit 0
 fi
fi' > /usr/local/bin/test_login.sh
            chmod +x /usr/local/bin/test_login.sh
            sed -i '6iaccount    required     pam_exec.so /usr/local/bin/test_login.sh' /etc/pam.d/sshd
            dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
            dnf install -y docker-ce --nobest
            systemctl start docker
            gpasswd -a duty docker
            echo 'polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") {
        if (action.lookup("unit") == "docker.service" && subject.user === "test1"){
            if (action.lookup("verb") == "restart") {
                return polkit.Result.YES;
            }
        }
    }
});' > /etc/polkit-1/rules.d/01-docker.rules
          SHELL

      end
      config.vbguest.auto_update = false
  end
end

