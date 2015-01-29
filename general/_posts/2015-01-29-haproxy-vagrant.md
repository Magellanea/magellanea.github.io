---
layout: post
title:  "Testing Load Balancer with Vagrant"
date:   2014-01-29 13:18:21
comments: true
---

In this post I will explain how can [haproxy](http://www.haproxy.org/) be used to test load balancer with vagrant by using a custom [plugin](https://github.com/Magellanea/vagrant_haproxy_provisioner)

This post follows that [article](https://www.digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-http-load-balancing-on-an-ubuntu-vps) but by using vagrant instead of a vps or a local machine.

- Create Vagrat file for nodes using a simple LAMP architecture
{% highlight ruby %}

Vagrant.configure(2) do |config|
  config.vm.box = "precise64"
  (1..n).each do |i|
    config.vm.define ("loadbalancer_node%d" % i) do |machine|
      options = {}
      machine.vm.network :public_network , options
      # change that to use chef/puppet
      machine.vm.provision "shell", path: "install_apache2.sh"
    end
  end
end

{% endhighlight %}

- We use a simple shell script for installing and configuring LAMP.
{% highlight bash %}
sudo apt-get update
sudo apt-get -y install apache2
sudo apt-get install -y php5 libapache2-mod-php5
sudo /etc/init.d/apache2 restart
sudo rm /var/www/index.html
sudo su
echo '<?php header("Content-Type: text/plain");echo "Server IP: ".$_SERVER["SERVER_ADDR"];echo "\nClient IP: ".$_SERVER["REMOTE_ADDR"];echo "\nX-Forwarded-for: ".$_SERVER["HTTP_X_FORWARDED_FOR"];?>' > /var/www/index.php
{% endhighlight %}


As you can see the final line replicates what was done in the reference article, it simply outputs the line so that we can know which node was responding.


- Finally we need to install the plugin to be used in provisioning, a simple way

{%highlight bash%}
git clone https://github.com/Magellanea/vagrant_haproxy_provisioner
## build the gem
bundle exec rake -T && bundle exec rake install
## in The VagrantFile directory install the plugin
vagrant plugin install path_to_gem/pkg/vagrant_haproxy_provisioner-0.0.1.gem
{%endhighlight%}

- You can now use the new provisioner with the original vagrant file which can now be as the following

{% highlight ruby %}
require 'json'

Vagrant.configure(2) do |config|
  # If the file doesn't exist, make ips automatics
  ips = File.exist?("ips.json") ? JSON.parse(open("ips.json").readline()) : nil
  n = ips.nil? ? 1 : ips.length
  config.vm.box = "precise64"
  (1..n).each do |i|
    config.vm.define ("loadbalancer_node%d" % i) do |machine|
      options = {}
      unless ips.nil?
        options[:ip] =  ips[i-1]
      end
      machine.vm.network :public_network , options
      # change that to use chef/puppet
      machine.vm.provision "shell", path: "install_apache2.sh"
    end
  end
  config.vm.define("loadbalancer") do |machine|
    machine.vm.network :public_network
    machine.vm.provision "haproxy_provisioner" do |p|
      p.ips = ips.nil? ? [] : ips
    end

  end
end
{% endhighlight %}

- The ```ips.json``` file is supposed to be created containing a ```JSON``` list of the ips that will be available to nodes.


- The final step is setting up the machines

{%highlight bash%}
vagrant up
{%endhighlight%}

- You can make an ssh connection to the machine named "loadbalancer" ```vagrant ssh loadbalancer``` to get it's ip.

- By using ```curl [loadbalancer-ip]``` you can track how the haproxy loads the page from a different host based on it's balancing method.
