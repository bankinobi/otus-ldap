# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :'ipaserver' => {
        :box_name => "centos/7",
        :ip => '192.168.50.20',
        :mem => '2048'
  },
  :'ipaclient' => {
        :box_name => "centos/7",
        :ip => '192.168.50.21',
        :mem => '1048'
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
        config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip]
          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", boxconfig[:mem]]
          end
          box.vm.provision "ansible" do |ansible|
            ansible.become = true
            ansible.verbose = "vv"
            ansible.playbook = "provision/provision.yml"
          end
      end
  end
end
