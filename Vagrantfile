# -*- mode: ruby -*-
# vi: set ft=ruby :

DEFAULT_MACHINE = {
  :domain => 'internal',
  :box => 'rockylinux/9/v9.5-20241118.0',
  :cpus => 2,
  :memory => 2048,
  :networks => {},
  :intnets => {},
  :forwarded_ports => [],
  :modifyvm => []
}

MACHINES = {
  :master => { :intnets => { :mysql => { ip: '192.168.11.150' } } },
  :slave  => { :intnets => { :mysql => { ip: '192.168.11.151' } } },
}

def provisioned?(host_name)
  return File.exist?('.vagrant/machines/' + host_name.to_s +
    '/virtualbox/action_provision')
end

Vagrant.configure('2') do |config|
  MACHINES.each do |host_name, host_config|
    host_config = DEFAULT_MACHINE.merge(host_config)
    config.vm.define host_name do |host|
      host.vm.box = host_config[:box]
      if not provisioned?(host_name)
        host.vm.host_name = host_name.to_s + '.' + host_config[:domain].to_s
      end

      host.vm.provider :virtualbox do |vb|
        vb.cpus = host_config[:cpus]
        vb.memory = host_config[:memory]

        if !host_config[:modifyvm].empty?
          vb.customize ['modifyvm', :id] + host_config[:modifyvm]
        end
      end

      host_config[:networks].each do |network_type, network_args|
        host.vm.network(network_type, **network_args)
      end
      host_config[:intnets].each do |name, intnet|
        intnet[:virtualbox__intnet] = name.to_s
        host.vm.network(:private_network, **intnet)
      end
      host_config[:forwarded_ports].each do |forwarded_port|
        host.vm.network(:forwarded_port, **forwarded_port)
      end

      if MACHINES.keys.last == host_name
        host.vm.provision :ansible do |ansible|
          ansible.playbook = 'playbook.yml'
          ansible.limit = 'all'
          ansible.compatibility_mode = '2.0'
          ansible.raw_arguments = ['--diff']
        end
      end

      host.vm.synced_folder '.', '/vagrant', disabled: true
    end
  end
end
