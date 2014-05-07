#
# Squishymedia's Puppetized Vagrantfile
#

# GETTING STARTED
#
# Get your vm set up:
#   vagrant up
# Set up hosts file on your local machine:
#   127.0.0.1 projectname.local
# Sync the database (if using Drupal):
#   vagrant ssh
#   drush sql-sync @projectname-dev @projectname-local
# Sync uploaded files (if using Drupal):
#   drush rsync @projectname-dev:%files @projectname-local:%files
#
# Visit the site at http://projectname.local:3080/

# ------------------------------------
VAGRANTFILE_API_VERSION = "2"

hostname = %x[ hostname -f ]
username = %x[ whoami ]

# You might want to replace this with a literal project name.
project  = File.basename(File.dirname(__FILE__));

#
# Compute IP address and forwarded ports based on the project name.
#
# The IP address is in the 10.0.0.0/8 private subnet, with the last three
# digits determined by the lowest 24 bits of the CRC.
#
# The forwarded ports are numbered starting at 3000. The least significant two
# digits identify the service; the others are computed based on the modulus of
# the CRC. For example, 'myproject' will have its port forwardings in the range
# 10800-10899.
#
# The hope is that these calculations will reduce the possibility of port and
# subnet collisions so that many projects can run simultaneously on the same
# machine without requiring manual changes to the Vagrantfile.
#
require 'zlib'
require 'ipaddr'
crc = Zlib::crc32(project)
ip = (crc & 0x00ffffff) | (10 << 24)
ip = IPAddr.new(ip, Socket::AF_INET).to_s
$port_base = (crc % 500) * 100 + 3000

puts "HOLLA AT A VAGRANT:"
puts "  http://localhost:" + ($port_base + 80).to_s + "/"
puts "  mysql -h 127.0.0.1 -P " + ($port_base + 06).to_s + " -u root -p"

# Now we are ready to configure the box.
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box_url = "https://911fc3b8b8cc070da44b-76fd772d1c308d2aec785b792582b337.ssl.cf2.rackcdn.com/Centos-6.4-x86_64_puppet_2013-06-11.box"
  config.vm.box = "Centos-6.4-x86_64_puppet_2013-06-11"
  config.vm.provider 'virtualbox' do |vbox|
    vbox.customize ["modifyvm", :id, "--memory", 1024]
    vbox.customize ["modifyvm", :id, "--cpus", 2]

    # Use VirtualBox's builtin DNS proxy to avoid silly DNS issues when the
    # host has a DNS proxy (such as Ubuntu).
    # See: https://www.virtualbox.org/ticket/10864
    vbox.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]

    # fix "read-only filesystem" errors in Mac OS X
    # see: https://github.com/mitchellh/vagrant/issues/713
    vbox.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/v-root", "1"]
    vbox.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/server", "1"]
  end

  # NFS mount needs hostonly net
  # Docs: http://docs.vagrantup.com/v2/networking/private_network.html
  config.vm.network :private_network, ip: ip

  # Mount webroot.
  #
  # NFS shared folders are several orders of magnitude faster, but they don't
  # work on Windows hosts, they can require a little configuration, and they
  # require that vagrant run some tasks as root. If you don't want to use NFS,
  # try enabling it here.  For more information, see:
  #
  # http://docs-v1.vagrantup.com/v1/docs/nfs.html
  #
  # To disable NFS, set :nfs => false here.
  config.vm.synced_folder ".", "/server", type: 'nfs'

  # Forward SSH key agent over the 'vagrant ssh' connection
  config.ssh.forward_agent = true

  # Enable to launch a VirtualBox console
  #cnf.vm.boot_mode = :gui

  config.vm.host_name = "web.%s.%s" % [ project, hostname.strip.to_s ]

  # Stuff can be done before puppet.
  config.vm.provision :shell, :path => 'vagrant/pre-puppet.sh', :args => project

  # Docs: http://docs-v1.vagrantup.com/v1/docs/provisioners/puppet.html
  config.vm.provision :puppet do |puppet|
    # Load puppet manifests from vagrant/manifests
    puppet.manifests_path = "vagrant/manifests"
    puppet.manifest_file  = "default.pp"
    puppet.module_path    = "vagrant/modules"
    # Set $vagrant = 1 inside puppet manifests
    puppet.facter = {
      "vagrant" => "1",
      "vagrant_ssh_user" => username.strip.to_s,
    }
    puppet.options = "--hiera_config /server/vagrant/hiera.yaml"
    # Send "notice" to syslog
    puppet.options += " --logdest syslog"
    # Enable this to see the details of a puppet run
    #puppet.options += " --verbose --debug"
  end

  # Stuff can be done after puppet.
  config.vm.provision :shell, :path => 'vagrant/post-puppet.sh', :args => project

  config.vm.network :forwarded_port, guest: 80,   host: $port_base + 80
  config.vm.network :forwarded_port, guest: 3306, host: $port_base + 6
end

# vim: set ft=ruby
