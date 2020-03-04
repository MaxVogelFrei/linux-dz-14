Vagrant.configure("2") do |config|

  config.vm.define "web" do |w|
    w.vm.box = "centos/7"
    w.vm.box_version = "1905.1"
    w.vm.hostname = "web"
    w.vm.network "private_network", ip: "192.168.14.3"
    w.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
    end
    w.vm.provision "shell", inline: <<-SHELL
      yum install epel-release -y
      yum install nginx audispd-plugins -y
      cp /vagrant/conf/web_nginx.conf /etc/nginx/nginx.conf
      cp /vagrant/conf/web_rsyslog.conf /etc/rsyslog.conf
      cp /vagrant/conf/web_audisp-remote.conf /etc/audisp/audisp-remote.conf
      cp /vagrant/conf/web_au-remote.conf /etc/audisp/plugins.d/au-remote.conf
      echo "-w /etc/nginx/ -p rwa -k nginx_config_access" >> /etc/audit/rules.d/audit.rules
      shutdown -r now
    SHELL
  end

  config.vm.define "log" do |l|
    l.vm.box = "centos/7"
    l.vm.box_version = "1905.1"
    l.vm.hostname = "log"
    l.vm.network "private_network", ip: "192.168.14.4"
    l.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
    end
    l.vm.provision "shell", inline: <<-SHELL
      cp /vagrant/conf/log_rsyslog.conf /etc/rsyslog.conf
      cp /vagrant/conf/log_auditd.conf /etc/audit/auditd.conf
      shutdown -r now
    SHELL
  end

end
