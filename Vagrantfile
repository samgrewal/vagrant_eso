Vagrant.configure("2") do |hello|
  hello.vm.box = "centos/7"
  hello.vm.network "forwarded_port", guest: 22, host: 2225, host_ip: '127.0.0.1'
  hello.vm.network "forwarded_port", guest: 80, host: 8085, host_ip: '127.0.0.1'

  hello.vm.boot_timeout = "900"

  # use default synced folder . -> /vagrant
  hello.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "/vagrant/provision/run_playbook.yml"
    ansible.verbose = true
    ansible.install = true
  end
end
