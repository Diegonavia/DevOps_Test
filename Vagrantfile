Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"
  
  config.vm.network "forwarded_port", guest: 80, host: 80
  config.vm.network "forwarded_port", guest: 8080, host: 8080
    
  config.vm.provider "virtualbox" do |v|
      v.name = "PFG_VM"
      v.gui = true
  	v.memory = 1024
  end
  
  config.vm.provision "ansible_local", run: "always" do  |ansible|
    ansible.playbook = "initial-setup.yml"
  end

end
