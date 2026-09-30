

# TP02 

Disclaimer :  Artéfact lié au fonctionnement de mon adaptateur externe / vmware au reboot la mac de vbr10 fluctue


## Architecture / configuration

### 1. Mapping interfaces

| Rôle               | Interface | MAC               | Détail                                                                      |
| ------------------ | --------- | ----------------- | --------------------------------------------------------------------------- |
| NIC NAT / Internet | nic0    | 00:0c:29:53:b6:41 | Port du bridge vmbr0 (VMnet8, NAT)                                        |
| NIC LAB            | ens37   | 00:0c:29:53:b6:4b | Port du bridge vmbr1 (VMnet2, Pont sur l'Ethernet LAB)                    |
| NIC OOB            | aucune    |                   | Non utilisée                                                                |
| vmbr0           | bridge    | 00:0c:29:53:b6:41 | 192.168.200.11/24, gateway 192.168.200.2 : Internet et route par défaut |
| vmbr1         | bridge    | 00:0c:29:53:b6:4b | 10.100.12.14/24, **aucune gateway** : underlay inter-PVE                  |
| vmbr10         | bridge    | 16:56:c2:8b:49:1b | Local au nœud, bridge-ports none, aucune IP                               |

### 2. Plan d'adressage

Réseau LAB du groupe 12 : 10.100.12.0/24, sans gateway.

> Le plan du cours prévoit `.11` à `.13`. mon nœud utilise `.14` : `check-tp02.sh` a donc été lancé avec l'IP attendue en 3ᵉ argument (`./scripts/proxmox/check-tp02.sh 12 1 10.100.12.14`). Le rang `1` sert uniquement à passer la validation des arguments.

### Plan d'adressages des noeuds du groupe : 

Réseau LAB du groupe 12 : 10.100.12.0/24, sans gateway.

| Nœud    | IP LAB            |
| ------- | ----------------- |
| PVE01   | `10.100.12.11/24` |
| pve02 | `10.100.12.12/24` |
| PVE03   | `10.100.12.13/24` |
| PVE04   | `10.100.12.14/24` |

### 3. Configuration appliquée

Sauvegarde préalable : `cp /etc/network/interfaces /root/interfaces.before-tp02`.

`/etc/network/interfaces` :

```
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.200.11/24
        gateway 192.168.200.2
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0

iface ens37 inet manual

auto vmbr1
iface vmbr1 inet static
        address 10.100.12.14/24
        bridge-ports ens37
        bridge-stp off
        bridge-fd 0

auto vmbr10
iface vmbr10 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0

source /etc/network/interfaces.d/*
```

Côté VMware : VMnet2 en Pont sur la carte _dynabook USB Ethernet_ (pas « Automatique »), carte réseau 2 de la VM sur VMnet2.

### 4. Schéma

```
Internet ── VMnet8 (NAT 192.168.200.0/24, gw .2)
                │
              nic0 ── vmbr0 (192.168.200.11/24)  ← route par défaut

              ┌── vmbr1 (10.100.12.14/24, pas de gateway)
   ens37 ─────┤        │
 (VMnet2,     │        └── autres PVE du VLAN groupe 12 (via switch)
  Pont Eth.)  │
              └── vmbr10 (local, sans port)
```

## Preuves utiles

### Sauvegarde
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# ls -l /root/interfaces.before-tp02
-rw-r--r-- 1 root root 226 Sep 29 09:01 /root/interfaces.before-tp02

### Routage

```
root@pve04:~# ip route
default via 192.168.200.2 dev vmbr0 proto kernel onlink
10.100.12.0/24 dev vmbr1 proto kernel scope link src 10.100.12.14
192.168.200.0/24 dev vmbr0 proto kernel scope link src 192.168.200.11
```

La route par défaut passe par `vmbr0` / NAT : c'est le seul chemin vers Internet. `vmbr1` n'a qu'une route connectée vers le réseau LAB, car il n'a pas de gateway. `ip route show default` ne renvoie qu'une seule ligne, et rien sur `vmbr1`.

### Interfaces et ports du bridge

```text
root@pve04:~# ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
nic0             UP
ens37            UP
vmbr0            UP             192.168.200.11/24 fe80::20c:29ff:fe53:b641/64
vmbr1            UP             10.100.12.14/24 fe80::20c:29ff:fe53:b64b/64
vmbr10           UNKNOWN        fe80::9411:56ff:fe98:aa4f/64

root@pve04:~# bridge link
2: nic0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master vmbr0 state forwarding priority 32 cost 100
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master vmbr1 state forwarding priority 32 cost 100
```

vmbr10 n'apparaît pas dans bridge link : il n'a aucun port. Son état UNKNOWN est normal, un bridge sans port n'a pas de lien physique à signaler.

### Connectivité

```text
root@pve04:~# ping -c 3 10.100.12.13
64 bytes from 10.100.12.13: icmp_seq=1 ttl=64 time=5.15 ms
64 bytes from 10.100.12.13: icmp_seq=2 ttl=64 time=5.43 ms
64 bytes from 10.100.12.13: icmp_seq=3 ttl=64 time=5.26 ms
3 packets transmitted, 3 received, 0% packet loss

root@pve04:~# ping -c 3 1.1.1.1
3 packets transmitted, 3 received, 0% packet loss
```

Le LAB et Internet fonctionnent en parallèle, par des chemins différents.

### ARP / Neighbor (`ip neigh`) : IP ↔ MAC

```text
root@pve04:~# ip neigh show dev vmbr1
10.100.12.12 FAILED
10.100.12.13 lladdr 00:0c:29:50:bb:86 DELAY
```

### FDB du bridge (`bridge fdb show br vmbr1`) : MAC ↔ port

```text
52:54:00:9e:40:02 dev ens37 master vmbr1
04:42:1a:01:50:41 dev ens37 master vmbr1
00:0c:29:50:bb:86 dev ens37 master vmbr1
00:50:b6:f7:ae:6e dev ens37 master vmbr1
40:c2:ba:3e:89:bc dev ens37 master vmbr1
00:0c:29:53:b6:4b dev ens37 vlan 1 master vmbr1 permanent
00:0c:29:53:b6:4b dev ens37 master vmbr1 permanent
```

(Les entrées multicast `33:33:…` et `01:00:5e:…` sont omises.) La MAC du camarade `00:0c:29:50:bb:86` a été apprise sur le port `ens37`. `00:0c:29:53:b6:4b` est la MAC locale, marquée `permanent`. Les quatre autres MAC sont d'autres machines du segment.

### Capture ARP pendant le diagnostic (avant correction)

```text
09:17:12.599891 00:0c:29:53:b6:4b > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.100.12.13 tell 10.100.12.14, length 28
09:17:13.644879 00:0c:29:53:b6:4b > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.100.12.13 tell 10.100.12.14, length 28
09:17:14.665201 00:0c:29:53:b6:4b > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.100.12.13 tell 10.100.12.14, length 28
```

Les requêtes partaient, sans aucune réponse ni trafic reçu d'une autre machine.
### Capture ICMP (`tcpdump -eni vmbr1 icmp`)

Axel (`10.100.12.13`) pingue mon nœud (`10.100.12.14`) :

```text
root@pve04:~# tcpdump -eni vmbr1 icmp
10:06:10.649392 00:0c:29:50:bb:86 > 00:0c:29:53:b6:4b, ethertype IPv4 (0x0800), length 98: 10.100.12.13 > 10.100.12.14: ICMP echo request, id 15162, seq 1, length 64
10:06:10.649441 00:0c:29:53:b6:4b > 00:0c:29:50:bb:86, ethertype IPv4 (0x0800), length 98: 10.100.12.14 > 10.100.12.13: ICMP echo reply, id 15162, seq 1, length 64
10:06:11.648434 00:0c:29:50:bb:86 > 00:0c:29:53:b6:4b, ethertype IPv4 (0x0800), length 98: 10.100.12.13 > 10.100.12.14: ICMP echo request, id 15162, seq 2, length 64
10:06:11.648496 00:0c:29:53:b6:4b > 00:0c:29:50:bb:86, ethertype IPv4 (0x0800), length 98: 10.100.12.14 > 10.100.12.13: ICMP echo reply, id 15162, seq 2, length 64
8 packets captured
```

Lecture : la MAC source de la requête (`00:0c:29:50:bb:86`, Axel) est la MAC destination de la réponse, et inversement pour la nôtre (`00:0c:29:53:b6:4b`). Les deux MAC sont celles de `ip neigh` et de `bridge fdb`. Le trafic reste en L2 sur `vmbr1`, sans passer par une gateway.
### Validation

```text
root@pve04:~# ./scripts/proxmox/check-tp02.sh 12 1 10.100.12.14
==============================================
       NOVACORP - TP02 VALIDATOR
==============================================
Expected LAB IP: 10.100.12.14

[PASS] vmbr0 exists
[PASS] Default route detected
[PASS] Default route is not on vmbr1
[PASS] vmbr1 exists
[PASS] Expected LAB IP found on vmbr1: 10.100.12.14
[PASS] vmbr1 has 1 bridge port(s): ens37
[PASS] vmbr1 is UP
[PASS] Internet connectivity OK
[PASS] vmbr10 exists
[PASS] vmbr10 has no physical bridge port
[PASS] Connected LAB route detected: 10.100.12.0/24 proto kernel scope link src 10.100.12.14

----------------------------------------------
PASS : 11
WARN : 0
FAIL : 0
STATUS: READY FOR TP03
```

### Troubleshooting : test de panne du LAB

Avant la panne : `ping 10.100.12.13` et `ping 1.1.1.1` répondent. Panne provoquée en décochant _Connecté_ sur la carte réseau 2 de la VM.

| Fonction                                           | Fonctionne ? | Pourquoi ?                                           |
| -------------------------------------------------- | -----------: | ---------------------------------------------------- |
| Internet PVE                                       |          oui | Chemin `vmbr0` → `nic0` → VMnet8, indépendant du LAB |
| `apt update`                                       |          oui | Passe par Internet, donc par `vmbr0`                 |
| GUI via réseau NAT (`https://192.168.200.11:8006`) |          oui | IP portée par `vmbr0`                                |
| ping PVE voisin via `vmbr1`                        |          non | Le seul chemin vers le voisin est la NIC LAB         |
| `vmbr10` local                                     |          oui | Bridge sans port, ne dépend d'aucune carte           |

Conclusion : une panne du réseau LAB n'entraîne pas de panne Internet, les chemins sont distincts (`vmbr0` contre `vmbr1`).


- Avant la panne, le LAB (`10.100.12.13`) et Internet répondent.
- Carte LAB déconnectée : le ping vers le voisin échoue (100 % de perte, puis `Destination Host Unreachable` une fois l'entrée ARP expirée), alors qu'Internet, `apt update` et `vmbr10` fonctionnent. Les chemins sont donc indépendants (`vmbr0` contre `vmbr1`).
- `ens37` reste affiché `UP` pendant la panne : VMware ne propage pas la coupure à l'état du lien virtuel. L'état d'une interface ne suffit donc pas, il faut vérifier le trafic réel.
- Après reconnexion, le ping repasse (premier paquet à ~1 s, le temps de rétablir l'ARP).

Sorties observées : 

```
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# ping -c 3 10.100.12.13
ping -c 3 1.1.1.1
PING 10.100.12.13 (10.100.12.13) 56(84) bytes of data.
64 bytes from 10.100.12.13: icmp_seq=1 ttl=64 time=3.64 ms
64 bytes from 10.100.12.13: icmp_seq=2 ttl=64 time=5.26 ms
64 bytes from 10.100.12.13: icmp_seq=3 ttl=64 time=5.07 ms

--- 10.100.12.13 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2009ms
rtt min/avg/max/mdev = 3.636/4.654/5.259/0.724 ms
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=128 time=9.44 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=128 time=10.9 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=128 time=11.2 ms

--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2018ms
rtt min/avg/max/mdev = 9.439/10.518/11.167/0.768 ms
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# ping -c 3 10.100.12.13
ping -c 3 1.1.1.1
apt update
ip -br link show ens37
ip link show vmbr10
PING 10.100.12.13 (10.100.12.13) 56(84) bytes of data.

--- 10.100.12.13 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2055ms

PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=128 time=10.8 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=128 time=9.40 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=128 time=10.9 ms

--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2008ms
rtt min/avg/max/mdev = 9.403/10.361/10.913/0.680 ms
Hit:1 http://security.debian.org/debian-security trixie-security InRelease
Hit:2 http://deb.debian.org/debian trixie InRelease
Hit:3 http://deb.debian.org/debian trixie-updates InRelease
Hit:4 http://download.proxmox.com/debian/pve trixie InRelease
181 packages can be upgraded. Run 'apt list --upgradable' to see them.
ens37            UP             00:0c:29:53:b6:4b <BROADCAST,MULTICAST,UP,LOWER_UP>
6: vmbr10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 16:56:c2:8b:49:1b brd ff:ff:ff:ff:ff:ff
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# ping -c 3 10.100.12.13
PING 10.100.12.13 (10.100.12.13) 56(84) bytes of data.
From 10.100.12.14 icmp_seq=1 Destination Host Unreachable
From 10.100.12.14 icmp_seq=2 Destination Host Unreachable
From 10.100.12.14 icmp_seq=3 Destination Host Unreachable

--- 10.100.12.13 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2057ms
pipe 3
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# ping -c 3 10.100.12.13
PING 10.100.12.13 (10.100.12.13) 56(84) bytes of data.
64 bytes from 10.100.12.13: icmp_seq=1 ttl=64 time=1078 ms
64 bytes from 10.100.12.13: icmp_seq=2 ttl=64 time=12.9 ms
64 bytes from 10.100.12.13: icmp_seq=3 ttl=64 time=3.45 ms

--- 10.100.12.13 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2071ms
rtt min/avg/max/mdev = 3.450/364.652/1077.558/504.115 ms, pipe 2
root@pve04:~/M1-Virtualisation-Cloud-Datacenter#
```

### Réponses aux questions obligatoires

**Question 1 : NIC physique, Linux Bridge, interface IP**

Une NIC (physique ou virtuelle) est l'interface qui envoie et reçoit les trames Ethernet, ici `nic0` ou `ens37`. Un Linux Bridge est un switch L2 logiciel (`vmbr0`, `vmbr1`) : il relie plusieurs ports (NIC, VM, conteneurs, interface de l'hôte) et apprend les MAC. Une interface IP est le point où l'hôte porte une adresse IP de couche 3. Sous Proxmox, l'IP est portée par le bridge, la NIC n'étant qu'un de ses ports.

**Question 2 : pourquoi l'IP est sur `vmbr1` et non sur `ens37`**

Une fois `ens37` membre du bridge, c'est le bridge qui décide de la commutation des trames. Si l'IP restait sur `ens37`, le trafic de l'hôte contournerait le bridge, et les VM branchées sur `vmbr1` ne partageraient pas le même segment L2 que l'hôte. En portant l'IP, `vmbr1` fait de l'hôte un participant du segment comme les VM, et évite la même adresse sur deux interfaces.

**Question 3 : pourquoi `vmbr1` n'a pas de gateway**

`vmbr1` est un underlay interne entre PVE : il ne mène à aucun routeur ni à Internet. Une gateway sur `vmbr1` créerait une seconde route par défaut, en conflit avec celle de `vmbr0`, et risquerait d'envoyer le trafic Internet vers un réseau sans sortie. La seule route par défaut doit rester sur `vmbr0` (NAT). Pour le LAB, la route connectée `10.100.12.0/24 dev vmbr1` suffit.

**Question 4 : pourquoi `vmbr10` de PVE01 et de PVE02 ne sont pas reliés**

Un Linux Bridge est local à un nœud : deux bridges portant le même nom sur deux machines sont deux switches indépendants. Ils ne sont reliés par aucun port physique (`bridge-ports none`), donc aucune trame ne passe de l'un à l'autre. Un Linux Bridge local n'est pas un réseau distribué : pour cela il faut un support commun (VLAN sur l'underlay) ou de l'overlay (VXLAN/SDN).

**Question 5 : commande pour la table MAC du bridge**

```bash
bridge fdb show br vmbr1
```

**Question 6 : `ip neigh` contre `bridge fdb`**

`ip neigh` affiche la table de voisinage de l'hôte, qui associe une **IP à une MAC** (ARP en IPv4, Neighbor Discovery en IPv6). `bridge fdb` affiche la Forwarding Database du bridge, qui associe une **MAC à un port**. Dans nos sorties : `10.100.12.13 → 00:0c:29:50:bb:86` (`ip neigh`), puis `00:0c:29:50:bb:86 → ens37` (`bridge fdb`).

### Questions du corps du TP

**ARP : quelle couche OSI, et pourquoi un échec pointe plus bas qu'un problème de routage**

ARP travaille principalement à la couche 2 (liaison), en faisant le lien avec la couche 3 : il résout une IP en MAC par un broadcast. Une résolution ARP échouée signifie que la trame n'a pas atteint la cible ou que celle-ci n'a pas répondu, alors que le routage n'est même pas encore en jeu. Le problème est donc plus bas qu'une erreur de route : câble, VLAN, pont ou cible éteinte.

## Difficultés rencontrées

- **ARP sans réponse** : après l'application de `vmbr1`, le ping vers un camarade renvoyait `Destination Host Unreachable`, avec `ip neigh` en `FAILED`. Un `tcpdump -eni vmbr1` montrait nos requêtes `ARP who-has` partir sans jamais recevoir de trafic d'une autre machine. La configuration était pourtant correcte : IP, port du bridge, absence de gateway et route connectée. Le diagnostic L3 (routes correctes) puis L2 (ARP sans réponse) a montré que le problème se situait hors de Proxmox.
- **Cause** : le pont VMware (VMnet2 en Pont sur l'adaptateur USB Ethernet) n'avait pas été pris en compte par la VM en cours d'exécution. Un **redémarrage de la VM** a corrigé le problème : le ping a ensuite répondu et la FDB a appris la MAC du camarade.
- **Rang hors plan** : `check-tp02.sh` n'accepte que les rangs 1 à 3, alors que notre IP est `.14`. Solution : passer l'IP explicitement en 3ᵉ argument, sans modifier le script.
- **Configuration VMware** : `VMnet2` était initialement en _Hôte uniquement_ et la carte 2 de la VM sur _Pont (Automatique)_ au lieu de `VMnet2`. Correction : `VMnet2` en Pont sur la carte Ethernet exacte, comme le demande le PRE-LAB.

## Ce que nous avons compris

- Un **Linux Bridge est un switch L2 logiciel** : l'IP de l'hôte se place sur le bridge, et la NIC n'en est que le port.
- Le chemin **Internet** (`vmbr0` → `nic0` → VMnet8) est distinct du chemin **LAB** (`vmbr1` → `ens37` → VMnet2 → switch) : une panne de l'un n'implique pas une panne de l'autre.
- La **route par défaut** ne doit exister qu'une fois, sur le réseau NAT. `vmbr1` n'a aucune gateway.
- **Diagnostiquer par couches** : L3 (adressage et routes), L2 (ARP, `ip neigh`, `bridge fdb`), L1 (lien, câble, pont VMware). Un ARP sans réponse pointe sous la couche routage.
- `ip neigh` (IP ↔ MAC) et `bridge fdb` (MAC ↔ port) se complètent : le premier dit quelle MAC porte l'IP, le second sur quel port elle a été apprise.
- Un état `UP` sur une carte virtuelle ne prouve pas que le lien physique fonctionne. Il faut regarder le trafic réellement échangé.
- Un `vmbr10` local sans port n'est pas partagé entre nœuds : il faudra VLAN ou VXLAN/SDN pour un réseau de VM commun à plusieurs PVE.
- Avant toute modification réseau sur un hyperviseur, **sauvegarder** `/etc/network/interfaces` et garder la console VMware pour le rollback.


### Bonus Expert 1

```
# 1. Avant : l'entrée d'Axel est dans la FDB
root@pve04:~# bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86
00:0c:29:50:bb:86 dev ens37 master vmbr1

# 2. Suppression de l'entrée, puis nouvelle lecture
root@pve04:~# bridge fdb del 00:0c:29:50:bb:86 dev ens37 master
root@pve04:~# bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86
(aucune sortie : l'entrée n'existe plus)

# 3. Trafic vers Axel
root@pve04:~# ping -c 3 10.100.12.13
64 bytes from 10.100.12.13: icmp_seq=1 ttl=64 time=3.55 ms
64 bytes from 10.100.12.13: icmp_seq=2 ttl=64 time=3.33 ms
64 bytes from 10.100.12.13: icmp_seq=3 ttl=64 time=3.88 ms
3 packets transmitted, 3 received, 0% packet loss

# 4. Après : l'entrée est réapprise
root@pve04:~# bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86
00:0c:29:50:bb:86 dev ens37 master vmbr1
```

```
root@pve04:~/M1-Virtualisation-Cloud-Datacenter# 
bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86


bridge fdb del 00:0c:29:50:bb:86 dev ens37 master
bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86 


ping -c 3 10.100.12.13


bridge fdb show br vmbr1 | grep 00:0c:29:50:bb:86
00:0c:29:50:bb:86 dev ens37 master vmbr1
PING 10.100.12.13 (10.100.12.13) 56(84) bytes of data.
64 bytes from 10.100.12.13: icmp_seq=1 ttl=64 time=3.55 ms
64 bytes from 10.100.12.13: icmp_seq=2 ttl=64 time=3.33 ms
64 bytes from 10.100.12.13: icmp_seq=3 ttl=64 time=3.88 ms

--- 10.100.12.13 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2022ms
rtt min/avg/max/mdev = 3.333/3.587/3.884/0.226 ms
00:0c:29:50:bb:86 dev ens37 master vmbr1
```

