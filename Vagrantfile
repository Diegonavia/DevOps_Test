Vagrant.configure(2) do |config|
  config.vm.network "public_network"

  config.vm.box = "centos/7"
  
  config.vm.network "forwarded_port", guest: 80, host: 80
  config.vm.network "forwarded_port", guest: 8080, host: 8080
    
  config.vm.provider "virtualbox" do |v|
      v.name = "PFG_VM"
      v.memory = 1024
  end
  
  config.vm.provision "ansible_local", run: "always" do  |ansible|
    ansible.playbook = "initial-setup.yml"
  end

  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "app-provision.yml"
  end

end
