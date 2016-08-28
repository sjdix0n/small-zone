# basic network layout for a small compute zone

       +---------------+          +-----------------+
       |       vsrx1   |__________|    vsrx2        |
       +---------------+          +-------+---------+
               |                          |
               |                          |
               swp1                       swp1   
               |                          |
               |                          |
               |                          |
+--------------+--------+  swp40 +--------+--------------+
|        sw1-leaf1      | _______|    sw2-leaf2          |
|                       |  swp50 |                       |
++-----+----------------+        +-+---------------------+
 |     |                           |
 |     swp11                       |
 |     |   +---------swp11---------+
 |     |   |
 |     |   |
 |     |   |
 | +---+---+-+
 | |   node1 |
 | |         |
 | +---------+
 |
 |
 |
 |                    +-----------+
 |                    | vyos1     |
 +-------swp10--------+           |
                      +-----------+

Basic small compute zone network design, two switches with trunk between them, uplinking to PE routers, vsrx used for testing, prod uses MX routers. Each hypervisor, (node) uplinks to each switch, uses vswitch options to load traffic between switches.   Switches also route node mgmt traffic using vrrp for HA.  vyos1 is a test server emulating path frames from a VM on a node would take to PE routers. VPLS used on PE's for HA between them.
