Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  
  config.vm.synced_folder ".", "/vagrant", type: "rsync"


  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbooks/setup.yml"
    ansible.inventory_path = "inventory"
    ansible.limit = "all"
  end

  config.vm.network "forwarded_port", guest: 8080, host: 8080
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  config.vm.network "forwarded_port", guest: 8086, host: 8086
   
  
 

end

