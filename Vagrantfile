# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant::Config.run do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.box_url = "ubuntu/trusty64" 
  config.vm.provision :shell, :inline => "sudo apt-get install curl -y"
  config.vm.provision :shell, :inline => "sudo apt-get install vim -y"

  config.vm.provision :shell, :inline => "sudo add-apt-repository ppa:brightbox/ruby-ng"
  config.vm.provision :shell, :inline => "sudo apt-get update"
  config.vm.provision :shell, :inline => "sudo apt-get install ruby2.3 -y"
  config.vm.provision :shell, :inline => "sudo apt-get install ruby-dev -y"
  config.vm.provision :shell, :inline => "sudo apt-get install ruby2.3-dev -y"
  config.vm.provision :shell, :inline => "sudo apt-get install zlib1g-dev -y"
  config.vm.provision :shell, :inline => "sudo gem install bundle"
  config.vm.provision :shell, :inline => "sudo gem install github-pages"
  config.vm.provision :shell, :inline => "sudo apt-get install git -y"

  config.vm.provision :shell, :inline => "echo All done, go vagrant ssh!"
  config.vm.provision :shell, :inline => "echo 'Serve Jekyll: cd /vagrant/ ; jekyll serve --host 0.0.0.0'"

  config.vm.forward_port 4000, 4000 # Jekyll
 
end

Vagrant.configure("2") do |config|

config.vm.provider :virtualbox do |virtualbox|
            
virtualbox.customize ["modifyvm", :id, "--memory", "1024"]     
                 
end
end
