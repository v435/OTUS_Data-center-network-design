# OS:
```
sysctl -w net.ipv4.conf.all.forwarding=1
```

# /etc/network/interfaces:
```
network:
  ethernets:
    e1:
      match:
        name: ens4
      set-name: e1
      addresses: [ 10.99.1.1/30 ]
    e2:
      match:
        name: ens5
      set-name: e2
      addresses: [ 10.99.2.1/30 ]
  bridges:
    lo0:
      addresses: [ 10.1.2.1/32 ]
  vrfs:
    VRF1:
      table: 1000
      interfaces: [ e1 ]
    VRF2:
      table: 2000
      interfaces: [ e2 ]

  version: 2
```

# FRR
```
frr version 10.0.1
frr defaults traditional
hostname GW-1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
ip prefix-list VRF1_CLIENT_NETS seq 5 permit 10.101.0.0/16 le 24
ip prefix-list VRF2_CLIENT_NETS seq 5 permit 10.102.0.0/16 le 24
!
vrf VRF1
exit-vrf
!
vrf VRF2
exit-vrf
!
router bgp 65099
 bgp router-id 10.1.2.1
exit
!
router bgp 65099 vrf VRF1
 bgp router-id 10.1.2.1
 no bgp ebgp-requires-policy
 neighbor 10.99.1.2 remote-as 65003
 neighbor 10.99.1.2 bfd
 neighbor 10.99.1.2 password OTUS
 neighbor 10.99.1.2 timers 3 9
 !
 address-family ipv4 unicast
  aggregate-address 10.102.0.0/16 summary-only
  neighbor 10.99.1.2 route-map VRF1_EXPORT out
  import vrf route-map VRF1_IMPORT
  import vrf VRF2
 exit-address-family
exit
!
router bgp 65099 vrf VRF2
 bgp router-id 10.1.2.1
 no bgp ebgp-requires-policy
 neighbor 10.99.2.2 remote-as 65003
 neighbor 10.99.2.2 bfd
 neighbor 10.99.2.2 password OTUS
 neighbor 10.99.2.2 timers 3 9
 !
 address-family ipv4 unicast
  aggregate-address 10.101.0.0/16 summary-only
  neighbor 10.99.2.2 as-override
  neighbor 10.99.2.2 route-map VRF2_EXPORT out
  import vrf route-map VRF2_IMPORT
  import vrf VRF1
 exit-address-family
exit
!
route-map VRF1_IMPORT permit 10
 match ip address prefix-list VRF2_CLIENT_NETS
 match source-vrf VRF2
exit
!
route-map VRF2_IMPORT permit 10
 match ip address prefix-list VRF1_CLIENT_NETS
 match source-vrf VRF1
exit
!
route-map VRF1_EXPORT permit 10
 match ip address prefix-list VRF2_CLIENT_NETS
 set as-path exclude all
exit
!
route-map VRF2_EXPORT permit 10
 match ip address prefix-list VRF1_CLIENT_NETS
 set as-path exclude all
exit
!
```