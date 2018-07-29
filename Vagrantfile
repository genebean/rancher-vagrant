# -*- mode: ruby -*-
# vi: set ft=ruby :

BOX_IMAGE = "genebean/centos-7-docker-ce"
NODE_COUNT = 3 # Minimum one node
RANCHER_PASSWORD = 'password'
RANCHER_CLUSTER_NAME = 'playground'
USE_STOCK_DOCKER = true # change to false to use Docker CE
UPDATE_DOCKER_CE = true

Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
    vb.linked_clone = true
  end

  config.vm.define "rancherMaster", primary: true do |master|
    master.vm.provider :virtualbox do |vb|
        vb.memory = 2048
        vb.cpus = 2
    end
    master.vm.box = BOX_IMAGE
    master.vm.hostname = "master.rancher.local"
    master.vm.network :private_network, ip: "192.168.34.10"
    if USE_STOCK_DOCKER
      master.vm.provision "shell", inline: <<-EOC
        yum -y remove docker-ce
        yum -y install docker
        systemctl enable docker
        systemctl start docker
      EOC
    elsif UPDATE_DOCKER_CE
      master.vm.provision "shell", inline: 'yum -y upgrade docker-ce'
    end
    master.vm.provision "docker" do |d|
      d.run "rancher-master",
        image: "rancher/rancher",
        restart: "unless-stopped",
        args: "-p 192.168.34.10:80:80 -p 192.168.34.10:443:443 --network=host"
    end
    master.vm.provision "shell", inline: <<-EOC
      set -e
      yum -y install jq
      while ! curl --output /dev/null --silent --insecure --head --fail  https://localhost/ping; do sleep 10 && \
      echo -n "waiting rancher to come up..."; done;

      # Login
      LOGINRESPONSE=`curl -s 'https://127.0.0.1/v3-public/localProviders/local?action=login' -H 'content-type: application/json' --data-binary '{"username":"admin","password":"admin"}' --insecure`
      LOGINTOKEN=`echo $LOGINRESPONSE | jq -r .token`

      # Change password
      curl -s 'https://127.0.0.1/v3/users?action=changepassword' -H 'content-type: application/json' -H "Authorization: Bearer $LOGINTOKEN" --data-binary '{"currentPassword":"admin","newPassword":"#{RANCHER_PASSWORD}"}' --insecure

      # Create API key
      APIRESPONSE=`curl -s 'https://127.0.0.1/v3/token' -H 'content-type: application/json' -H "Authorization: Bearer $LOGINTOKEN" --data-binary '{"type":"token","description":"automation"}' --insecure`

      # Extract and store token
      APITOKEN=`echo $APIRESPONSE | jq -r .token`

      RANCHER_SERVER='https://master.rancher.local'
      curl -s 'https://127.0.0.1/v3/settings/server-url' -H 'content-type: application/json' -H "Authorization: Bearer $APITOKEN" -X PUT --data-binary '{"name":"server-url","value":"'$RANCHER_SERVER'"}' --insecure

      # Create cluster
      CLUSTERRESPONSE=`curl -s 'https://127.0.0.1/v3/cluster' -H 'content-type: application/json' -H "Authorization: Bearer $APITOKEN" --data-binary '{"type":"cluster","nodes":[],"rancherKubernetesEngineConfig":{"ignoreDockerVersion":true},"name":"#{RANCHER_CLUSTER_NAME}"}' --insecure`

      # Extract clusterid to use for generating the docker run command
      CLUSTERID=`echo $CLUSTERRESPONSE | jq -r .id`

      # Roles
      ROLEFLAGS="--etcd --controlplane --worker"

      # Generate token (clusterRegistrationToken) and extract nodeCommand
      AGENTCOMMAND=`curl -s 'https://127.0.0.1/v3/clusterregistrationtoken' -H 'content-type: application/json' -H "Authorization: Bearer $APITOKEN" --data-binary '{"type":"clusterRegistrationToken","clusterId":"'$CLUSTERID'"}' --insecure | jq -r .nodeCommand`

      # Show the command
      echo 'set -x' > /agent_command.sh
      echo "${AGENTCOMMAND} ${ROLEFLAGS}" >> /agent_command.sh
      chmod a+x /agent_command.sh
    EOC
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define "rancherNode#{i}" do |node|
      node.vm.box = BOX_IMAGE
      node.vm.hostname = "node#{i}.rancher.local"
      node.vm.network :private_network, ip: "192.168.34.#{i + 10}"
      if USE_STOCK_DOCKER
        node.vm.provision "shell", inline: <<-EOC
          yum -y remove docker-ce
          yum -y install docker
          systemctl enable docker
          systemctl start docker
        EOC
      elsif UPDATE_DOCKER_CE
        node.vm.provision "shell", inline: 'yum -y upgrade docker-ce'
      end
      node.vm.provision "shell", inline: <<-EOC
        set -e
        yum -y install sshpass
        sshpass -p 'vagrant' scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@master.rancher.local:/agent_command.sh /agent_command.sh
        sed -i '${s/$/ --address 192.168.34.#{i + 10} --internal-address 192.168.34.#{i + 10}/}' /agent_command.sh
        /agent_command.sh
      EOC
    end
  end

end
