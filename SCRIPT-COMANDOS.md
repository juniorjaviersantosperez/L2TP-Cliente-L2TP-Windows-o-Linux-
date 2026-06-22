# Script de Configuración Completa
# VPN Client-to-Site Multipunto — IPSec IKEv1 con L2TP

---

## PARTE 1 — CONFIGURACIÓN BÁSICA DE EQUIPOS

### R-SERVIDOR-VPN

```cisco
enable
configure terminal

hostname R-SERVIDOR-VPN

interface ethernet0/0
 description LAN
 ip address 10.15.99.1 255.255.255.0
 no shutdown

interface ethernet0/1
 description WAN-ISP
 ip address 200.0.0.1 255.255.255.252
 no shutdown

end
write memory
```

---

### ISP

```cisco
enable
configure terminal

hostname ISP

interface ethernet0/0
 description WAN-SERVIDOR-VPN
 ip address 200.0.0.2 255.255.255.252
 no shutdown

interface ethernet0/2
 description HACIA-R5
 ip address 10.0.99.1 255.255.255.0
 no shutdown

interface ethernet0/1
 description HACIA-R6
 ip address 10.0.15.1 255.255.255.0
 no shutdown

end
write memory
```

---

### R5

```cisco
enable
configure terminal

hostname R5

interface ethernet0/0
 description HACIA-ISP
 ip address 10.0.99.2 255.255.255.0
 no shutdown

interface ethernet0/1
 description HACIA-SW1-LINUX8
 ip address 192.168.99.1 255.255.255.0
 no shutdown

end
write memory
```

---

### R6 (R11)

```cisco
enable
configure terminal

hostname R6

interface ethernet0/0
 description HACIA-ISP
 ip address 10.0.15.2 255.255.255.0
 no shutdown

interface ethernet0/1
 description HACIA-LINUX9
 ip address 192.168.15.1 255.255.255.0
 no shutdown

end
write memory
```

---

## PARTE 2 — ENRUTAMIENTO

### R-SERVIDOR-VPN

```cisco
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 200.0.0.2
end
write memory
```

### ISP

```cisco
enable
configure terminal
ip route 192.168.99.0 255.255.255.0 10.0.99.2
ip route 192.168.15.0 255.255.255.0 10.0.15.2
ip route 10.15.99.0 255.255.255.0 200.0.0.1
end
write memory
```

### R5

```cisco
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 10.0.99.1
end
write memory
```

### R6

```cisco
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 10.0.15.1
end
write memory
```

---

## PARTE 3 — CONFIGURACIÓN VPN EN R-SERVIDOR-VPN

```cisco
enable
configure terminal

! AAA y usuarios
aaa new-model
aaa authentication ppp VPN-AUTH local
aaa authorization network VPN-AUTHOR local
username kali1 secret cisco
username kali2 secret cisco

! Pool de IPs para clientes VPN
ip local pool VPN-POOL 192.168.99.10 192.168.99.50

! IKEv1 Phase 1
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key cisco address 0.0.0.0 0.0.0.0

! IPSec Phase 2
crypto ipsec transform-set L2TP-SET esp-aes 256 esp-sha256-hmac
 mode transport
crypto ipsec profile L2TP-PROFILE
 set transform-set L2TP-SET

! VPDN L2TP
vpdn enable
vpdn-group L2TP-SERVER
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

! Virtual Template PPP
interface Virtual-Template1
 ip unnumbered ethernet0/0
 peer default ip address pool VPN-POOL
 ppp authentication ms-chap-v2 VPN-AUTH
 ppp encrypt mppe auto

! Crypto Map dinámico
crypto dynamic-map DYN-MAP 10
 set transform-set L2TP-SET
 reverse-route
crypto map VPN-MAP 10 ipsec-isakmp dynamic DYN-MAP

! Aplicar Crypto Map a interfaz WAN
interface ethernet0/1
 crypto map VPN-MAP

end
write memory
```

---

## PARTE 4 — CONFIGURACIÓN CLIENTES KALI LINUX

### Configuración de red — Linux8

```bash
sudo ip addr add 192.168.99.2/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.99.1
```

### Configuración de red — Linux9

```bash
sudo ip addr add 192.168.15.15/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.15.1
```

### Instalación de dependencias (ambos Kali)

```bash
sudo apt update
sudo apt install -y strongswan xl2tpd ppp
```

### /etc/ipsec.conf — Linux8

```bash
sudo bash -c 'cat > /etc/ipsec.conf << EOF
config setup
    charondebug="ike 2, knl 2, cfg 2"

conn L2TP-PSK
    keyexchange=ikev1
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    type=transport
    left=%defaultroute
    leftid=192.168.99.2
    leftprotoport=17/1701
    right=200.0.0.1
    rightid=200.0.0.1
    rightprotoport=17/1701
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
    ikelifetime=86400s
    lifetime=3600s
EOF'
```

### /etc/ipsec.conf — Linux9

```bash
sudo bash -c 'cat > /etc/ipsec.conf << EOF
config setup
    charondebug="ike 2, knl 2, cfg 2"

conn L2TP-PSK
    keyexchange=ikev1
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    type=transport
    left=%defaultroute
    leftid=192.168.15.15
    leftprotoport=17/1701
    right=200.0.0.1
    rightid=200.0.0.1
    rightprotoport=17/1701
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
    ikelifetime=86400s
    lifetime=3600s
EOF'
```

### /etc/ipsec.secrets (ambos Kali)

```bash
sudo bash -c 'cat > /etc/ipsec.secrets << EOF
%any 200.0.0.1 : PSK "cisco"
EOF'
```

### /etc/xl2tpd/xl2tpd.conf (agregar al final, ambos Kali)

```bash
sudo bash -c 'cat >> /etc/xl2tpd/xl2tpd.conf << EOF

[global]
port = 1701

[lac vpn-cisco]
lns = 200.0.0.1
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
EOF'
```

### /etc/ppp/options.l2tpd.client — Linux8

```bash
sudo bash -c 'cat > /etc/ppp/options.l2tpd.client << EOF
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
name kali1
password cisco
idle 1800
mtu 1410
mru 1410
defaultroute
usepeerdns
debug
EOF'
```

### /etc/ppp/options.l2tpd.client — Linux9

```bash
sudo bash -c 'cat > /etc/ppp/options.l2tpd.client << EOF
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
name kali2
password cisco
idle 1800
mtu 1410
mru 1410
defaultroute
usepeerdns
debug
EOF'
```

---

## PARTE 5 — LEVANTAR LA VPN (AMBOS KALI)

```bash
sudo systemctl restart strongswan-starter
sudo systemctl restart xl2tpd
sudo ipsec up L2TP-PSK
sleep 2
echo "c vpn-cisco" | sudo tee /var/run/xl2tpd/l2tp-control
sleep 3
ip addr show ppp0
```

---

## PARTE 6 — VERIFICACIÓN

### En R-SERVIDOR-VPN

```cisco
show crypto isakmp policy
show crypto isakmp sa
show crypto ipsec sa
show vpdn session
show ppp all
show ip local pool VPN-POOL
show run | section crypto
show run | section vpdn
show run | section aaa
```

### En cada Kali

```bash
sudo ipsec status
ip addr show ppp0
ping 10.15.99.1 -c 4
ping 10.15.99.2 -c 4
```
