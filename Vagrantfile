# Created by Topology-Converter v4.3.1
#    https://github.com/cumulusnetworks/topology_converter
#    using topology data from: ../../lab/small-zone/small-zone.dot
#
#    NOTE: in order to use this Vagrantfile you will need:
#       -Vagrant(v1.8.1+) installed: http://www.vagrantup.com/downloads
#       -Cumulus Plugin for Vagrant installed: $ vagrant plugin install vagrant-cumulus
#       -the "helper_scripts" directory that comes packaged with topology-converter.py
#       -Virtualbox installed: https://www.virtualbox.org/wiki/Downloads



# Check required plugins
REQUIRED_PLUGINS = %w(vagrant-cumulus)
exit unless REQUIRED_PLUGINS.all? do |plugin|
  Vagrant.has_plugin?(plugin) || (
    puts "The #{plugin} plugin is required. Please install it with:"
    puts "$ vagrant plugin install #{plugin}"
    false
  )
end

$script = <<-SCRIPT
if grep -q -i 'cumulus' /etc/lsb-release &> /dev/null; then
    echo "### RUNNING CUMULUS EXTRA CONFIG ###"
    source /etc/lsb-release
    if [[ $DISTRIB_RELEASE =~ ^2.* ]]; then
        echo "  INFO: Detected a 2.5.x Based Release"
        echo "  adding fake cl-acltool..."
        echo -e "#!/bin/bash\nexit 0" > /bin/cl-acltool
        chmod 755 /bin/cl-acltool

        echo "  adding fake cl-license..."
        cat > /bin/cl-license <<'EOF'
#! /bin/bash
#-------------------------------------------------------------------------------
#
# Copyright 2013 Cumulus Networks, Inc.  All rights reserved
#

URL_RE='^http://.*/.*'

#Legacy symlink
if echo "$0" | grep -q "cl-license-install$"; then
    exec cl-license -i $*
fi

LIC_DIR="/etc/cumulus"
PERSIST_LIC_DIR="/mnt/persist/etc/cumulus"

#Print current license, if any.
if [ -z "$1" ]; then
    if [ ! -f "$LIC_DIR/.license.txt" ]; then
        echo "No license installed!" >&2
        exit 20
    fi
    cat "$LIC_DIR/.license.txt"
    exit 0
fi

#Must be root beyond this point
if (( EUID != 0 )); then
   echo "You must have root privileges to run this command." 1>&2
   exit 100
fi

#Delete license
if [ x"$1" == "x-d" ]; then
    rm -f "$LIC_DIR/.license.txt"
    rm -f "$PERSIST_LIC_DIR/.license.txt"

    echo License file uninstalled.
    exit 0
fi

function usage {
    echo "Usage: $0 (-i (license_file | URL) | -d)" >&2
    echo "    -i  Install a license, via stdin, file, or URL." >&2
    echo "    -d  Delete the current installed license." >&2
    echo >&2
    echo " cl-license prints, installs or deletes a license on this switch." >&2
}

if [ x"$1" != 'x-i' ]; then
    usage
    exit 100
fi
shift
if [ ! -f "$1" ]; then
    if [ -n "$1" ]; then
        if ! echo "$1" | grep -q "$URL_RE"; then
            usage
            echo "file $1 not found or not readable." >&2
            exit 100
        fi
    fi
fi

function clean_tmp {
    rm $1
}

if [ -z "$1" ]; then
    LIC_FILE=`mktemp lic.XXXXXX`
    trap "clean_tmp $LIC_FILE" EXIT
    echo "Paste license text here, then hit ctrl-d" >&2
    cat >$LIC_FILE
else
    if echo "$1" | grep -q "$URL_RE"; then
        LIC_FILE=`mktemp lic.XXXXXX`
        trap "clean_tmp $LIC_FILE" EXIT
        if ! wget "$1" -O $LIC_FILE; then
            echo "Couldn't download $1 via HTTP!" >&2
            exit 10
        fi
    else
        LIC_FILE="$1"
    fi
fi

/usr/sbin/switchd -lic "$LIC_FILE"
SWITCHD_RETCODE=$?
if [ $SWITCHD_RETCODE -eq 99 ]; then
    more /usr/share/cumulus/EULA.txt
    echo "I (on behalf of the entity who will be using the software) accept"
    read -p "and agree to the EULA  (yes/no): "
    if [ "$REPLY" != "yes" -a "$REPLY" != "y" ]; then
        echo EULA not agreed to, aborting. >&2
        exit 2
    fi
elif [ $SWITCHD_RETCODE -ne 0 ]; then
    echo '******************************' >&2
    echo ERROR: License file not valid. >&2
    echo ERROR: No license installed. >&2
    echo '******************************' >&2
    exit 1
fi

mkdir -p "$LIC_DIR"
cp "$LIC_FILE" "$LIC_DIR/.license.txt"
chmod 644 "$LIC_DIR/.license.txt"

mkdir -p "$PERSIST_LIC_DIR"
cp "$LIC_FILE" "$PERSIST_LIC_DIR/.license.txt"
chmod 644 "$PERSIST_LIC_DIR/.license.txt"

echo License file installed.
echo Reboot to enable functionality.
EOF
        chmod 755 /bin/cl-license

        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/init.d/rename_eth_swp /etc/init.d/rename_eth_swp.backup

        echo "  Replacing fake switchd"
        rm -rf /usr/bin/switchd
        cat > /usr/sbin/switchd <<'EOF'
#!/bin/bash
PIDFILE=$1
LICENSE_FILE=/etc/cumulus/.license
RC=0

# Make sure we weren't invoked with "-lic"
if [ "$PIDFILE" == "-lic" ]; then
  if [ "$2" != "" ]; then
        LICENSE_FILE=$2
  fi
  if [ ! -e $LICENSE_FILE ]; then
    echo "No license file." >&2
    RC=1
  fi
  RC=0
else
  tail -f /dev/null & CPID=$!
  echo -n $CPID > $PIDFILE
  wait $CPID
fi

exit $RC
EOF
        chmod 755 /usr/sbin/switchd

        cat > /etc/init.d/switchd <<'EOF'
#! /bin/bash
### BEGIN INIT INFO
# Provides:          switchd
# Required-Start:
# Required-Stop:
# Should-Start:
# Should-Stop:
# X-Start-Before
# Default-Start:     S
# Default-Stop:
# Short-Description: Controls fake switchd process
### END INIT INFO

# Author: Kristian Van Der Vliet <kristian@cumulusnetworks.com>
#
# Please remove the "Author" lines above and replace them
# with your own name if you copy and modify this script.

PATH=/sbin:/bin:/usr/bin

NAME=switchd
SCRIPTNAME=/etc/init.d/$NAME
PIDFILE=/var/run/switchd.pid

. /lib/init/vars.sh
. /lib/lsb/init-functions

do_start() {
  echo "[ ok ] Starting Cumulus Networks switch chip daemon: switchd"
  /usr/sbin/switchd $PIDFILE 2>/dev/null &
}

do_stop() {
  if [ -e $PIDFILE ];then
    kill -TERM $(cat $PIDFILE)
    rm $PIDFILE
  fi
}

do_status() {
  if [ -e $PIDFILE ];then
    echo "[ ok ] switchd is running."
  else
    echo "[FAIL] switchd is not running ... failed!" >&2
    exit 3
  fi
}

case "$1" in
  start|"")
	log_action_begin_msg "Starting switchd"
	do_start
	log_action_end_msg 0
	;;
  stop)
  log_action_begin_msg "Stopping switchd"
  do_stop
  log_action_end_msg 0
  ;;
  status)
  do_status
  ;;
  restart)
	log_action_begin_msg "Re-starting switchd"
  do_stop
	do_start
	log_action_end_msg 0
	;;
  reload|force-reload)
	echo "Error: argument '$1' not supported" >&2
	exit 3
	;;
  *)
	echo "Usage: $SCRIPTNAME [start|stop|restart|status]" >&2
	exit 3
	;;
esac

:
EOF
        chmod 755 /etc/init.d/switchd
        echo "### Rebooting to Apply Remap..."
        reboot

    elif [[ $DISTRIB_RELEASE =~ ^3.* ]]; then
        echo "  INFO: Detected a 3.x Based Release"

        echo "  Disabling default remap on Cumulus VX..."
        mv -v /etc/hw_init.d/S10rename_eth_swp.sh /etc/S10rename_eth_swp.sh.backup
        echo "### Applying Remap without Reboot..."
        /home/vagrant/apply_udev.py --apply
        echo "### Performing IFRELOAD to Apply any Latent Interface Config..."
        ifreload -a       
    fi
    echo "### DONE ###"
else
    echo "### Rebooting to Apply Remap..."
    reboot
fi
SCRIPT

Vagrant.configure("2") do |config|
  wbid = 1472251425

  config.vm.provider "virtualbox" do |v|
    v.gui=false

  end

  #Generating Ansible Host File at following location:
  #    ./.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "./helper_scripts/empty_playbook.yml"
#    ansible.playbook = "./playbook.yml"
  end


  ##### DEFINE VM for leaf1 #####
  config.vm.define "leaf1" do |device|
    device.vm.hostname = "leaf1" 
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "1472251425_leaf1"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true

    # NETWORK INTERFACES
      # link for swp1 --> vsrx1:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net15", auto_config: false , :mac => "44383900001e"
      
      # link for swp10 --> server1:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net1", auto_config: false , :mac => "443839000002"
      
      # link for swp11 --> node1:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net3", auto_config: false , :mac => "443839000006"
      
      # link for swp12 --> node1:swp3
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net2", auto_config: false , :mac => "443839000004"
      
      # link for swp13 --> node1:swp5
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net11", auto_config: false , :mac => "443839000016"
      
      # link for swp14 --> node1:swp7
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net5", auto_config: false , :mac => "44383900000a"
      
      # link for swp40 --> leaf2:swp40
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net14", auto_config: false , :mac => "44383900001b"
      
      # link for swp50 --> leaf2:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net4", auto_config: false , :mac => "443839000007"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous error, fixing this now...') || true;"



    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
      device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
      device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"

      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:1e swp1"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:02 swp10"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:06 swp11"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:04 swp12"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:16 swp13"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:0a swp14"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:1b swp40"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:07 swp50"

      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"



    
    # Run Any Platform Specific Code and Apply the interface Re-map 
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script


  end

  ##### DEFINE VM for leaf2 #####
  config.vm.define "leaf2" do |device|
    device.vm.hostname = "leaf2" 
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "1472251425_leaf2"
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true

    # NETWORK INTERFACES
      # link for swp1 --> vsrx2:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net10", auto_config: false , :mac => "443839000014"
      
      # link for swp10 --> server1:eth2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net8", auto_config: false , :mac => "443839000010"
      
      # link for swp11 --> node1:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net13", auto_config: false , :mac => "44383900001a"
      
      # link for swp12 --> node1:swp4
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net12", auto_config: false , :mac => "443839000018"
      
      # link for swp13 --> node1:swp6
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net7", auto_config: false , :mac => "44383900000e"
      
      # link for swp14 --> node1:swp8
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net9", auto_config: false , :mac => "443839000012"
      
      # link for swp40 --> leaf1:swp40
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net14", auto_config: false , :mac => "44383900001c"
      
      # link for swp50 --> leaf1:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net4", auto_config: false , :mac => "443839000008"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all']
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all']
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous error, fixing this now...') || true;"



    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
      device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
      device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"

      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:14 swp1"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:10 swp10"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:1a swp11"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:18 swp12"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:0e swp13"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:12 swp14"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:1c swp40"
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:08 swp50"

      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"



    
    # Run Any Platform Specific Code and Apply the interface Re-map 
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script


  end

  ##### DEFINE VM for vsrx1 #####
  config.vm.define "vsrx1" do |vsrx1|
    vsrx1.vm.hostname = "vsrx1" 
    vsrx1.vm.box = "juniper/ffp-12.1X47-D20.7-packetmode"
    vsrx1.vm.provider "virtualbox" do |v|
      v.name = "1472251425_vsrx1"
      v.memory = 1024
    end
    vsrx1.vm.network :forwarded_port, guest: 22, host: 12203, id: 'ssh'
    # link for eth1 --> leaf1:swp1
    vsrx1.vm.network "private_network", 
                      virtualbox__intnet: "#{wbid}_net15"
    # link for eth3 --> vsrx2:eth3
    vsrx1.vm.network "private_network", 
                      virtualbox__intnet: "#{wbid}_net6",
                      ip: "10.0.3.1",
                      netmask: "255.255.255.252"
    end
    
        


  ##### DEFINE VM for vsrx2 #####
  config.vm.define "vsrx2" do |vsrx2|
    vsrx2.vm.hostname = "vsrx2" 
    vsrx2.vm.box = "juniper/ffp-12.1X47-D20.7-packetmode"
    vsrx2.vm.provider "virtualbox" do |v|
      v.name = "1472251425_vsrx2"
      v.memory = 1024
    end
    vsrx2.vm.network :forwarded_port, guest: 22, host: 12204, id: 'ssh'
    # link for eth1 --> leaf2:swp1
    vsrx2.vm.network "private_network", 
                      virtualbox__intnet: "#{wbid}_net10"
    # link for eth3 --> vsrx1:eth3
    vsrx2.vm.network "private_network", 
                      virtualbox__intnet: "#{wbid}_net6",
                      ip: "10.0.3.2",
                      netmask: "255.255.255.252"
    end


  ##### DEFINE VM for server1 #####
#  config.vm.define "server1" do |device|
#    device.vm.hostname = "server1" 
#    device.vm.box = "boxcutter/ubuntu1404"
#    device.vm.provider "virtualbox" do |v|
#      v.name = "1472251425_server1"
#      v.memory = 512
#    end
#    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
#    device.vm.synced_folder ".", "/vagrant", disabled: true
#      # Shorten Boot Process - Applies to Ubuntu Only - remove \"Wait for Network\"
#      device.vm.provision :shell , inline: "sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf || true"

#    # NETWORK INTERFACES
#      # link for eth1 --> leaf1:swp10
#      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net1", auto_config: false , :mac => "443839000001"
#      
#      # link for eth2 --> leaf2:swp10
#      device.vm.network "private_network", virtualbox__intnet: "#{wbid}_net8", auto_config: false , :mac => "44383900000f"
      

#    device.vm.provider "virtualbox" do |vbox|
#      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
#      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all']
#      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
#    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
#    device.vm.provision :shell , inline: "(grep -q 'mesg n' /root/.profile 2>/dev/null && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous error, fixing this now...') || true;"



    # Run the Config specified in the Node Attributes
#    device.vm.provision :shell , path: "./helper_scripts/extra_server_config.sh"


    # Install Rules for the interface re-map
#      device.vm.provision "file", source: "./helper_scripts/apply_udev.py", destination: "/home/vagrant/apply_udev.py"
#      device.vm.provision :shell , inline: "chmod 755 /home/vagrant/apply_udev.py"

#      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:01 eth1"
#      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -a 44:38:39:00:00:0f eth2"

#      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -vm "
#      device.vm.provision :shell , inline: "/home/vagrant/apply_udev.py -s"



    
    # Run Any Platform Specific Code and Apply the interface Re-map 
    #   (may or may not perform a reboot depending on platform)
#    device.vm.provision :shell , :inline => $script


#  end
 
  config.vm.define :vyos1 do |vyos1|
    vyos1.vm.box = "higebu/vyos-1.1.7-amd64"
    vyos1.vm.network "private_network", ip: "192.168.1.10", virtualbox__intnet: "#{wbid}_net1"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  end




end
