# This Vagrantfile sets up a VirtualBox VM for testing the ansible scripts in.
VAGRANTFILE_API_VERSION = '2'

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.define 'vm1' do |box|
    box.vm.box = 'ubuntu/bionic64'
    box.vm.provider :virtualbox do |virtualbox|
      virtualbox.customize ['modifyvm', :id, '--memory', '2048']
    end
    box.vm.network 'forwarded_port', guest: 80, host: 8080
  end
end
