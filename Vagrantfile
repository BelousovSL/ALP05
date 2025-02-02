# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'
Vagrant.configure("2") do |config|
   config.vm.define "belousovRaid" do |belousovRaid|
      belousovRaid.vm.box = "bento/ubuntu-24.04"      
      belousovRaid.vm.host_name = "belousovRaid"
      belousovRaid.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = "2"
      end

      # Понимаю что в цикле создавать одну запись странно, но вот честно Ruby, не язык моей мечты. 
      (1..1).each do |i|
        belousovRaid.vm.disk :disk, size: "1GB", name: "disk-#{i}"
      end     

   end
end