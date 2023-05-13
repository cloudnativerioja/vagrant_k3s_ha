Vagrant.configure("2") do |config|
  (1..3).each do |i|
    config.vm.define "k3s-#{i}" do |k3s|
      k3s.vm.box = "ubuntu/focal64"
      k3s.vm.network "private_network", ip: "192.168.56.1#{i}", :name => 'vboxnet0', :adapter => 2
      k3s.vm.provider "virtualbox" do |vb|
          vb.memory = 1024
          vb.cpus = 2
          vb.name = "k3s-#{i}"
      end
      k3s.vm.hostname = "k3s-#{i}"
      k3s.vm.provision "shell", inline: <<-EOF
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

        sudo apt update -y
        sudo apt install jq net-tools -y

        curl -sLS https://get.k3sup.dev | sh

        mkdir /home/vagrant/.kube/
        chown vagrant:vagrant /home/vagrant/.kube/

        sudo su -c 'echo "192.168.56.11 k3s-1" >> /etc/hosts'
        sudo su -c 'echo "192.168.56.12 k3s-2" >> /etc/hosts'
        sudo su -c 'echo "192.168.56.13 k3s-3" >> /etc/hosts'
      EOF
      if i == 1
        k3s.vm.provision "shell", inline: <<-EOF
          ssh-keygen -t rsa -f /home/vagrant/.ssh/id_rsa -N ""
          chown vagrant:vagrant /home/vagrant/.ssh/id_rsa
          cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
          cp -p /home/vagrant/.ssh/id_rsa.pub /vagrant/id_rsa.pub
          cp -p /home/vagrant/.ssh/id_rsa /vagrant/id_rsa

          sudo su vagrant -c 'k3sup install --ip 192.168.56.1#{i} --tls-san 192.168.56.10 --cluster --k3s-channel latest --k3s-extra-args "--node-ip 192.168.56.1#{i} --advertise-address 192.168.56.1#{i} --node-external-ip 192.168.56.1#{i} --disable servicelb --disable traefik" --local-path /home/vagrant/.kube/config --user vagrant'


          mkdir -p /var/lib/rancher/k3s/server/manifests
          curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
          export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
          ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip manifest daemonset \
            --interface enp0s8 \
            --address 192.168.56.10 \
            --inCluster \
            --taint \
            --controlplane \
            --services \
            --arp \
            --leaderElection | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

          sed -i "s~https://[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+:[0-9]\+~https://192.168.56.10:6443~g" /home/vagrant/.kube/config
          echo "Esperando a que el cluster esté ready..."
          sleep 30
          su vagrant -c 'kubectl get nodes'
          cp -p /home/vagrant/.kube/config /vagrant/kubeconfig
        EOF
      else
        k3s.vm.provision "shell", inline: <<-EOF
          cp -p /vagrant/id_rsa.pub /home/vagrant/.ssh/id_rsa.pub
          cp -p /vagrant/id_rsa /home/vagrant/.ssh/id_rsa
          cat /vagrant/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
          cp -p /vagrant/kubeconfig /home/vagrant/.kube/config
          echo "configurando k3s-#{i}"
          su vagrant -c 'k3sup join --ip 192.168.56.1#{i} --server-ip 192.168.56.10 --server --k3s-channel latest --user vagrant --k3s-extra-args "--node-ip 192.168.56.1#{i} --advertise-address 192.168.56.1#{i} --node-external-ip 192.168.56.1#{i} --disable servicelb --disable traefik"'
          sleep 30
          su vagrant -c 'kubectl get nodes'
        EOF
      end
    end
  end
end