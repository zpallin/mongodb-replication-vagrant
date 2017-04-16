# -*- mode: ruby -*-
# vi: set ft=ruby :

vms = [
  {:name => 'mongo3', :ip => '192.168.33.13'},
  {:name => 'mongo2', :ip => '192.168.33.12'},
  {:name => 'mongo1', :ip => '192.168.33.11'} # last one becomes primary
]

hostname_add = vms.map{|x|"#{x[:ip]} #{x[:name]}"}.join('\n')
hostname_match = 'mongo1'

service_init = <<-SERVICE
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongodb
ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

[Install]
WantedBy=multi-user.target
SERVICE

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-16.04"
  vms.each do |vm|
    postprovision = ""
    if vm == vms.last
      postprovision = <<-POSTPROVISION
      mongo #{vm[:ip]}:27017 --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"#{vm[:name]}:27017"}]})'
      sleep 5
      mongo #{vm[:ip]}:27017 --eval 'rs.add("mongo2:27017")'
      mongo #{vm[:ip]}:27017 --eval 'rs.add("mongo3:27017")'
      POSTPROVISION
    end


    config.vm.define vm[:name] do |server|
      server.vm.network 'private_network', ip: vm[:ip]
      server.vm.hostname = vm[:name]
      server.vm.provider :virtualbox do |vb|
        vb.name = vm[:name]
      end
      server.vm.provision :shell, inline: <<-SHELL
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6
        echo "deb [ arch=amd64 ] http://repo.mongodb.org/apt/ubuntu precise/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list
        sudo apt update -y
        sudo apt install -y mongodb-org
        if grep -v #{hostname_match} /etc/hosts; then 
          sudo echo -en "#{hostname_add}" >> /etc/hosts; 
        fi
        sudo echo -en "#{service_init}" > /etc/systemd/system/mongodb.service
        sudo service mongodb start
        sudo sed -i 's/bindIp:.*/bindIp:\ #{vm[:ip]}/' /etc/mongod.conf
        sudo sed -i 's/\#replication:/replication:\\\n\ \ replSetName:\ rs0/' /etc/mongod.conf
        sudo service mongodb restart
        #{postprovision}
      SHELL
    end
  end
end

