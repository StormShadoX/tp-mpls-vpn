# Commandes du TP 2 : VPN MPLS

Les commandes sont à coller **étape par étape**, routeur par routeur, en validant les tests de chaque étape avant de passer à la suivante. Toutes les commandes de configuration se tapent en mode privilégié (`enable`).

Hypothèse d'adressage des interfaces PE-CE : SP1 = `192.168.12.1`, SP3 = `192.168.45.1`. Vérifier l'adresse réelle avec `show run interface Serial1/1` **avant** de taper `ip vrf forwarding`, car l'adresse est supprimée.

---

## 0. Prérequis : backbone du TP 1

Vérifications à faire avant de commencer.

**SP1**
```
enable
show mpls ldp neighbor
show ip route ospf
ping 3.3.3.3 source 1.1.1.1
show run interface Serial1/1
```

**SP3**
```
enable
show mpls ldp neighbor
show ip route ospf
ping 1.1.1.1 source 3.3.3.3
show run interface Serial1/1
```

Attendu : un voisin LDP (SP2) en état `Oper`, routes OSPF vers les loopbacks, pings à 100 %. `mpls ip` n'est pas activé sur les interfaces PE-CE.

---

## Étape 1 : VRF Client_A sur les PE

**SP1**
```
conf t
ip vrf Client_A
 rd 100:1
 route-target both 1:100
exit
interface Serial1/1
 ip vrf forwarding Client_A
 ip address 192.168.12.1 255.255.255.0
 no shutdown
end
```

**SP3**
```
conf t
ip vrf Client_A
 rd 100:1
 route-target both 1:100
exit
interface Serial1/1
 ip vrf forwarding Client_A
 ip address 192.168.45.1 255.255.255.0
 no shutdown
end
```

### Tests de l'étape 1

**SP1**
```
ping 192.168.12.2
ping vrf Client_A 192.168.12.2
show ip vrf
show ip vrf interfaces
```

**SIEGE**
```
ping 192.168.12.1
```

**SP3**
```
ping 192.168.45.2
ping vrf Client_A 192.168.45.2
show ip vrf
show ip vrf interfaces
```

**AGENCE**
```
ping 192.168.45.1
```

Attendu : le ping sans `vrf` échoue depuis le PE, avec `vrf Client_A` il réussit ; depuis le CE, le ping vers le PE réussit.

**Question du TP :** une route VPNv4 avec la RT `100:100` n'est **pas** importée, car la VRF n'importe que `1:100`.

---

## Étape 2 : EIGRP dans la VRF sur les PE

**SP1**
```
conf t
router eigrp 1
 address-family ipv4 vrf Client_A
  network 192.168.12.0
  no auto-summary
  autonomous-system 100
 exit-address-family
end
```

**SP3**
```
conf t
router eigrp 1
 address-family ipv4 vrf Client_A
  network 192.168.45.0
  no auto-summary
  autonomous-system 100
 exit-address-family
end
```

### Vérifications de l'étape 2

```
SP1# show ip eigrp vrf Client_A neighbors
SP3# show ip eigrp vrf Client_A neighbors
SP1# show ip route
SP3# show ip route
SP1# show ip route vrf Client_A
SP3# show ip route vrf Client_A
SIEGE# show ip route
AGENCE# show ip route
```

Attendu :
- chaque PE a son CE comme voisin EIGRP ;
- la table globale (`show ip route`) ne contient pas 192.168.12.0/24 ni 192.168.45.0/24 ;
- la table VRF contient des routes **D** uniquement du côté local ;
- SIEGE et SP1 n'ont pas de route vers le LAN AGENCE ; SP3 et AGENCE n'ont pas de route vers le LAN SIEGE (pas encore de BGP).

---

## Étape 3 : BGP VPNv4 et redistribution

### 3.1 Session BGP entre SP1 et SP3

**SP1**
```
conf t
router bgp 1
 neighbor 3.3.3.3 remote-as 1
 neighbor 3.3.3.3 update-source Loopback0
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community both
 exit-address-family
end
```

**SP3**
```
conf t
router bgp 1
 neighbor 1.1.1.1 remote-as 1
 neighbor 1.1.1.1 update-source Loopback0
 address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community both
 exit-address-family
end
```

**Vérification (SP1 et SP3)**
```
show bgp vpnv4 unicast all summary
```
La session doit être `Established` (un nombre dans la colonne State/PfxRcd). Attendre environ une minute.

### 3.2 Redistribution BGP et EIGRP

À coller seulement quand la session BGP est établie.

**SP1**
```
conf t
router eigrp 1
 address-family ipv4 vrf Client_A
  redistribute bgp 1 metric 1500 4000 200 10 1500
 exit-address-family
exit
router bgp 1
 address-family ipv4 vrf Client_A
  redistribute eigrp 100
 exit-address-family
end
```

**SP3**
```
conf t
router eigrp 1
 address-family ipv4 vrf Client_A
  redistribute bgp 1 metric 1500 4000 200 10 1500
 exit-address-family
exit
router bgp 1
 address-family ipv4 vrf Client_A
  redistribute eigrp 100
 exit-address-family
end
```

### Vérifications de l'étape 3

```
SP3# show ip route vrf Client_A
SP1# show ip route vrf Client_A
SIEGE# show ip route
AGENCE# show ip route
```

Attendu :
- SP3 : routes **B** vers 192.168.12.0/24 et 10.10.10.0/24 (site SIEGE) ;
- SP1 : routes **B** vers 192.168.45.0/24 et 192.168.1.0/24 (site AGENCE) ;
- SIEGE et AGENCE : routes **D** vers le site opposé (via SP1 et SP3).

---

## Étape 4 : tests de connectivité

### 4.1 Capture Wireshark sur SP2-SP3

Dans GNS3 : clic droit sur le lien SP2-SP3 > **Start capture** (type de liaison selon l'encapsulation), **avant** les pings.

Vérifier l'encapsulation du lien :
```
SP2# show interfaces Serial1/3 | include Encapsulation
```
`HDLC` donne Cisco HDLC, `PPP` donne PPP.

### 4.2 Pings

```
SIEGE# ping 192.168.1.1
AGENCE# ping 10.10.10.1
```

Variante depuis le LAN :
```
SIEGE# ping 192.168.1.1 source FastEthernet0/0
AGENCE# ping 10.10.10.1 source FastEthernet0/0
```

### 4.3 Wireshark

Filtre d'affichage :
```
mpls
```
Sur un paquet ICMP, déplier **MultiProtocol Label Switching** : deux labels empilés (transport LDP + VPN BGP).

### 4.4 Traceroutes

```
AGENCE# traceroute 10.10.10.1
SIEGE# traceroute 192.168.1.1
```

### 4.5 Label stacking

```
SP1# show mpls forwarding-table
SP3# show mpls forwarding-table
SP1# show bgp vpnv4 unicast all labels
SP3# show bgp vpnv4 unicast all labels
```

---

## Récupérer les captures de la VM GNS3 (PowerShell, Windows)

```
cd ~
ssh gns3@192.168.56.101 "ls -lt /opt/gns3/projects"
scp -r gns3@192.168.56.101:/opt/gns3/projects/UUID/project-files/captures .\captures
```
Remplacer `UUID` par l'identifiant du projet. Ne pas mettre `http://` dans l'adresse de `scp`.

---

## Publier sur GitHub

```
cd ~/tp-mpls-vpn
git init
git add .
git status
git commit -m "TP2 VPN MPLS : configs, captures et topologie"
git branch -M main
git remote add origin https://github.com/TON_UTILISATEUR/tp-mpls-vpn.git
git push -u origin main
```

Ne pas pousser la vidéo MP4 (limite de 100 Mo par fichier) : mettre son lien dans le README.
