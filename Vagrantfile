# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

class String
    def black;          "\e[30m#{self}\e[0m" end
    def red;            "\e[31m#{self}\e[0m" end
    def cyan;           "\e[36m#{self}\e[0m" end
end


engine_version=''
engine_mode='default'
proxy = ''


config = YAML.load_file(File.join(File.dirname(__FILE__), 'instances.yaml'))

base_box=config['environment']['base_box']
base_box_version=config['environment']['base_box_version']

boxes = config['boxes']

boxes_hostsfile_entries=""

rke2_cluster_ip=config['rke2']['rke2_cluster_ip']
rke2_cluster_fqdn=config['rke2']['rke2_cluster_fqdn']



########


boxes.each do |box|
  boxes_hostsfile_entries=boxes_hostsfile_entries+box['cluster_ip'] + ' ' +  box['name'] + '\n'
end

#puts boxes_hostsfile_entries

$update_hosts = <<SCRIPT
    echo "127.0.0.1 localhost" >/etc/hosts
    echo -e "#{boxes_hostsfile_entries}" |tee -a /etc/hosts
SCRIPT

puts '-------------------------------------------------------'

puts ' RKE2 Vagrant Environment'

# puts ' Engine Version: '+engine_version

puts '-------------------------------------------------------'

#$install_docker_engine = <<SCRIPT
#  curl -sSk $1 | sh
#  usermod -aG docker vagrant 2>/dev/null
#SCRIPT

$disable_swap = <<SCRIPT
    swapoff -a 
    sed -i '/swap/{ s|^|#| }' /etc/fstab
SCRIPT

$disable_firewall = <<SCRIPT
    systemctl disable --now firewalld
SCRIPT

$install_rke2server = <<SCRIPT
  curl -sfL https://get.rke2.io | sh -
  cat <<EOF > /etc/rancher/rke2/config.yaml
  advertise-address: $1
  write-kubeconfig-mode: "0644"
  tls-san:
    - "$2"
    - "$3"
  node-label:
    - "environment=vagrant"
    - "cluster=$3"
EOF
  systemctl enable --now rke2-server 
  #firewall-cmd --add-port 6443/tcp --permanent 
  #firewall-cmd --add-port 9345/tcp --permanent 
SCRIPT

$install_rke2agent = <<SCRIPT
  curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
  cp /tmp_deploying_stage/rke2-fisrt-server-token /etc/rancher/rke2/rke2-fisrt-server-token
  cat <<EOF >/etc/rancher/rke2/config.yaml
  advertise-address: $1
  server: https://$2:9345
  token-file: /etc/rancher/rke2/rke2-fisrt-server-token
EOF
  systemctl enable --now rke2-agent
SCRIPT

$configure_client = <<SCRIPT
  echo -e "export PATH=\$PATH:/var/lib/rancher/rke2/bin\nexport KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >/root/rke2_env.sh
SCRIPT

$get_rke2server_token = <<SCRIPT
  cp /var/lib/rancher/rke2/server/node-token  /tmp_deploying_stage/rke2-fisrt-server-token
SCRIPT

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if proxy != ''
        puts " Using proxy"
        config.proxy.http = proxy
        config.proxy.https = proxy
        config.proxy.no_proxy = "localhost,127.0.0.1"
    end
  end
  config.vm.box = base_box
  config.vm.box_version = base_box_version
  config.vm.synced_folder "tmp_deploying_stage/", "/tmp_deploying_stage",create:true
  config.vm.synced_folder "labs/", "/labs",create:true
  config.vm.synced_folder "labs/", "/labs",create:true
  boxes.each do |node|
    config.vm.define node['name'] do |config|
      config.vm.hostname = node['name']
      config.vm.provider "virtualbox" do |v|
        v.name = node['name']
        v.customize ["modifyvm", :id, "--memory", node['mem']]
        v.customize ["modifyvm", :id, "--cpus", node['cpu']]
	      v.customize ["modifyvm", :id, "--macaddress1", "auto"]
        #v.customize ["modifyvm", :id, "--nictype1", "Am79C973"]
        #v.customize ["modifyvm", :id, "--nictype2", "Am79C973"]
        #v.customize ["modifyvm", :id, "--nictype3", "Am79C973"]
        #v.customize ["modifyvm", :id, "--nictype4", "Am79C973"]

        v.customize ["modifyvm", :id, "--nicpromisc1", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]      
      end

      config.vm.network "private_network",
        ip: node['cluster_ip'],
        virtualbox__intnet: true


      config.vm.network "private_network",
        ip: node['hostonly_ip']

      # config.vm.network "public_network",
      # bridge: ["enp4s0","wlp3s0","enp3s0f1","wlp2s0"],
      # auto_config: true


      config.vm.provision :shell, :inline => $update_hosts

#      config.vm.provision "shell" do |s|
#       			s.name       = "Install Docker Engine from "+engine_download_url
#        		s.inline     = $install_docker_engine
#            s.args       = engine_download_url
#      end

      config.vm.provision :shell, :inline => $disable_swap
      config.vm.provision :shell, :inline => $disable_firewall

      if node['role'] == "control-plane"
        config.vm.provision "shell" do |s|
              s.name       = "Install RKE2 Server using 'curl' method"
              s.inline     = $install_rke2server
              s.args       = [ node['cluster_ip'],rke2_cluster_ip,rke2_cluster_fqdn ]
        end

        config.vm.provision "shell" do |s|
          s.name       = "Configure Kubernetes Client"
          s.inline     = $configure_client
        end

        config.vm.provision "shell" do |s|
          s.name       = "Get RKE2 Server token"
          s.inline     = $get_rke2server_token
        end      
      
        config.vm.network "forwarded_port", guest: 6443, host: 6443, auto_correct: true
      
      else
        config.vm.provision "shell" do |s|
          s.name       = "Install RKE2 Agent using 'curl' method"
          s.inline     = $install_rke2agent
          s.args       = [ node['cluster_ip'], rke2_cluster_ip ]
        end
        
        

      end

    end
  end

end
