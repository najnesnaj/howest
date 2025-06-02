Proxmox
==========


remove firewall from talos worker node

.. code-block:: bash


    root@pve:~# qm set 109 -net0 virtio,bridge=vmbr0,firewall=0
    update VM 109: -net0 virtio,bridge=vmbr0,firewall=0

setting up an extra network bridge vmbr1
-----------------------------------------

on lxc dedicated machine setup dhcp and routing

.. code-block:: bash


      apt install dnsmasq -y
      apt install iptables-persistent -y

      vi /etc/dnsmasq.conf 
      interface=eth0
      dhcp-range=10.10.10.100,10.10.10.200,12h  # DHCP range for Talos nodes
      dhcp-option=3,10.10.10.2                  # Gateway (this machine’s eth0 IP)
      dhcp-option=6,192.168.0.1                 # DNS (your home router’s DNS)

      systemctl restart dnsmasq

      echo 1 > /proc/sys/net/ipv4/ip_forward

      iptables -t nat -L -v
      ip route del default via 10.10.10.1 dev eth0
      ip route replace default via 192.168.0.1 dev eth1 metric 100

      iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth1 -j MASQUERADE
      ip route del default via 10.10.10.1 dev eth0
      ip route replace default via 192.168.0.1 dev eth1 metric 100



      vi /etc/netplan/01-netcfg.yaml


.. code-block:: yaml

    network:
      version: 2
      ethernets:
        eth0:
          dhcp4: true
          # Prevent DHCP from setting a default gateway if it conflicts
          dhcp4-overrides:
            use-routes: true
            use-dns: true
            route-metric: 2000  # High metric to prioritize eth1's default route
        eth1:
          dhcp4: false
          addresses:
            - 192.168.0.x/24  # Replace with your server's IP on this subnet
          routes:
            - to: 0.0.0.0/0
              via: 192.168.0.1
              metric: 100
            - to: 0.0.0.0/0
              via: 192.168.0.1
              metric: 1024
            - to: 192.168.0.0/24
              via: 0.0.0.0
              metric: 1024
            - to: 192.168.0.1
              via: 0.0.0.0
              metric: 1024
            - to: <gent.dnscache01-ip>
              via: 192.168.0.1
              metric: 1024
            - to: <gent.dnscache02-ip>
              via: 192.168.0.1
              metric: 1024

.. code-block:: bash

      # Apply the netplan configuration
      sudo netplan generate
      sudo netplan apply

      # Check the routing table
      ip route show

      # Check iptables rules
      iptables -t nat -L -v

      # Check dnsmasq status
      systemctl status dnsmasq

      # Check if the DHCP server is running and listening on the correct interface
      sudo systemctl status dnsmasq

      # Restart dnsmasq to apply changes
      sudo systemctl restart dnsmasq


      netplan apply


Kernel IP routing table
-----------------------

.. list-table::
      
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    default         192.168.0.1     0.0.0.0         UG    100    0        0 eth1
    default         192.168.0.1     0.0.0.0         UG    1024   0        0 eth1
    10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
    192.168.0.0     0.0.0.0         255.255.255.0   U     1024   0        0 eth1
    192.168.0.1     0.0.0.0         255.255.255.255 UH    1024   0        0 eth1
    gent.dnscache01 192.168.0.1     255.255.255.255 UGH   1024   0        0 eth1
    gent.dnscache02 192.168.0.1     255.255.255.255 UGH   1024   0        0 eth1

using the nodeport
------------------
192.168.0.251:30743



on my router/dhcp on 10.10.10.2 route port to cluster node IP

iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 30743 -j DNAT --to-destination 10.10.10.118:30743

so running nginx on kubernetes on 10.10.10.255 network is accessible from the outside


using the IP address
--------------------
traefik      LoadBalancer   10.102.122.212   10.10.10.50   80:32178/TCP,443:32318/TCP   75m   app.kubernetes.io/instance=traefik-default,app.kubernetes.io/name=traefik

So now I have to figure out how I can reach  10.10.10.50 from my 192.168.X.X network 

on the kubernetes cluster, traefik has been deployed as well as metallb.
iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -d 10.10.10.0/24 -j MASQUERADE
sh -c "iptables-save > /etc/iptables/rules.v4"

this has been added to th dnsmasq.conf

# Listen on the 192.168.0.251 interface
interface=eth1  # Replace with your 192.168.0.251 interface (check with `ip a`)
listen-address=192.168.0.251

# Forward other queries to upstream DNS (e.g., Google DNS)
server=8.8.8.8
server=8.8.4.4

# Optional: If LXC is your DHCP server, ensure DNS is offered
dhcp-option=6,192.168.0.251  # Tells DHCP clients to use this as DNS

modify dns config on laptop
---------------------------- 
/etc/resolv.conf

add : nameserver 192.168.0.251


access http://nginx.example.com/ on talos within 10.10.10.X from 192.168.X.X
-----------------------------------------------------------------------------
(configure metallb, traefik, nginx)


on laptop /etc/hosts : 10.10.10.50 nginx.example.com

on dhcp server (10.10.10.2)

iptables -A FORWARD -s 192.168.0.0/24 -d 10.10.10.0/24 -j ACCEPT
iptables -A FORWARD -s 10.10.10.0/24 -d 192.168.0.0/24 -j ACCEPT

.. code-block:: bash
  
    # Generated by iptables-save v1.8.7 on Thu Apr 10 13:32:59 2025
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A FORWARD -s 192.168.0.0/24 -d 10.10.10.0/24 -j ACCEPT
    -A FORWARD -s 10.10.10.0/24 -d 192.168.0.0/24 -j ACCEPT
    COMMIT
    # Completed on Thu Apr 10 13:32:59 2025
    # Generated by iptables-save v1.8.7 on Thu Apr 10 13:32:59 2025
    *nat
    :PREROUTING ACCEPT [6847:1975161]
    :INPUT ACCEPT [158:15156]
    :OUTPUT ACCEPT [25:2590]
    :POSTROUTING ACCEPT [25:2590]
    -A PREROUTING -i eth1 -p tcp -m tcp --dport 30743 -j DNAT --to-destination 10.10.10.118:30743
    -A POSTROUTING -s 10.10.10.0/24 -o eth1 -j MASQUERADE
    -A POSTROUTING -s 10.10.10.0/24 -o eth1 -j MASQUERADE
    -A POSTROUTING -s 10.10.10.0/24 -o eth1 -j MASQUERADE
    -A POSTROUTING -s 192.168.0.0/24 -d 10.10.10.0/24 -j MASQUERADE
    COMMIT
    # Completed on Thu Apr 10 13:32:59 2025
    # Check the iptables rules
    iptables -t nat -L -v
    iptables -L -v

    # Check the routing table
    ip route show

    # Check the network interfaces
    ip a