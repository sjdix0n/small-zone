# basic network layout for a small compute zone

  
See small-zone.png for diagram 

Basic small compute zone network design, two switches with trunk between them, uplinking to PE routers, vsrx used for testing, prod uses MX routers. Each hypervisor, (node) uplinks to each switch, uses vswitch options to load traffic between switches.   Switches also route node mgmt traffic using vrrp for HA.  vyos1 is a test server emulating path frames from a VM on a node would take to PE routers. VPLS used on PE's for HA between them.

requirements.
Ansible
 - ansible-galaxy install Juniper.junos
 - ansible-galaxy install cumulus.CumulusLinux
Vagrant 
 - vagrant plugin install vagrant-junos
 - vagrant plugin install vagrant-vyos
 - vagrant plugin install vagrant-host-shell
 
