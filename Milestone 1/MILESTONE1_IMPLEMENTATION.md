# Milestone 1 - Routage, Firewall et VPN site-to-site

Ce document decrit l'implementation complete du Milestone 1 pour le projet 504 Network Security. L'objectif est de construire le reseau de base dans GNS3, configurer le routage entre les zones, appliquer une politique firewall en whitelist, puis mettre en place un VPN IPsec entre R1 et R2 avec certificats PKI.

Topologie de reference: `Milestone 1/504 PROJECT_Milestone1 topo.png`.

## 1. Objectif du Milestone 1

Le Milestone 1 met en place:

- les reseaux externes `192.168.10.0/24` et `192.168.20.0/24`;
- les reseaux bureaux `10.10.10.0/24` et `10.10.20.0/24`;
- les reseaux serveurs `10.20.10.0/24` et `10.20.20.0/24`;
- le routage entre R1, R2, le firewall et R3;
- une politique firewall qui autorise uniquement le trafic necessaire;
- un VPN IPsec site-to-site entre R1 et R2;
- des tests de connectivite et de filtrage pour prouver le resultat.

## 2. Plan d'adressage

### 2.1 Clients

| Machine | Reseau | Adresse IP | Gateway |
| --- | --- | --- | --- |
| Remote-Employee | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.254` |
| Malicious PC | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.254` |
| PC-1 IT | `10.10.10.0/24` | `10.10.10.1` | `10.10.10.254` |
| PC-2 HR | `10.10.20.0/24` | `10.10.20.1` | `10.10.20.254` |

### 2.2 Routeurs et firewall

| Equipement | Interface | Adresse IP | Description |
| --- | --- | --- | --- |
| R1 | `f0/0` | `192.168.10.254/24` | Remote Employee |
| R1 | `f0/1` | `192.168.20.254/24` | Malicious PC |
| R1 | `f1/0` | `192.168.255.1/30` | Lien vers R2 |
| R1 | `f3/0` | `172.16.1.1/30` | Lien vers CA |
| R2 | `f0/0` | `192.168.255.2/30` | Lien vers R1 |
| R2 | `f0/1` | `10.0.1.1/30` | Lien vers firewall |
| R2 | `f1/0` | `10.10.10.254/24` | PC-1 IT |
| R2 | `f2/0` | `10.10.20.254/24` | PC-2 HR |
| R2 | `f3/0` | `172.16.2.1/30` | Lien vers CA |
| Firewall | `f0/0` | `10.0.1.2/30` | Cote R2 |
| Firewall | `f0/1` | `10.0.2.1/30` | Cote R3 |
| R3 | `f0/0` | `10.0.2.2/30` | Lien vers firewall |
| R3 | `f0/1` | `10.20.10.254/24` | Server1 |
| R3 | `f1/0` | `10.20.20.254/24` | Server2 |

### 2.3 Serveurs

| Machine | Adresse IP | Gateway |
| --- | --- | --- |
| Server1 | `10.20.10.1/24` | `10.20.10.254` |
| Server2 | `10.20.20.1/24` | `10.20.20.254` |

### 2.4 Autorite de certification

| Equipement | Interface | Adresse IP |
| --- | --- | --- |
| CA | `f0/0` | `172.16.1.2/30` |
| CA | `f0/1` | `172.16.2.2/30` |

## 3. Configuration des PC VPCS

Dans GNS3, les VPCS utilisent la syntaxe:

```text
ip <adresse> <masque> <passerelle>
save
```

### 3.1 Remote-Employee

```text
ip 192.168.10.1 255.255.255.0 192.168.10.254
save
```

### 3.2 Malicious PC

```text
ip 192.168.20.1 255.255.255.0 192.168.20.254
save
```

### 3.3 PC-1 IT

```text
ip 10.10.10.1 255.255.255.0 10.10.10.254
save
```

### 3.4 PC-2 HR

```text
ip 10.10.20.1 255.255.255.0 10.10.20.254
save
```

Important: dans GNS3, les IP des clients peuvent ne pas etre conservees apres reimport. Il faut verifier les VPCS au debut de chaque demo avec `show ip`.

## 4. Configuration des serveurs Debian/Linux

### 4.1 Server1

```bash
ip addr add 10.20.10.1/24 dev eth0
ip link set eth0 up
ip route add default via 10.20.10.254
ip route
```

### 4.2 Server2

```bash
ip addr add 10.20.20.1/24 dev eth0
ip link set eth0 up
ip route add default via 10.20.20.254
ip route
```

Si l'image Linux ne garde pas la configuration apres redemarrage, il faut refaire ces commandes ou configurer `/etc/network/interfaces`.

## 5. Configuration des routeurs Cisco

Avant les configurations VPN, regler l'horloge des routeurs pour eviter les problemes de certificat:

```text
enable
clock set 14:30:00 Mar 30 2026
```

### 5.1 R1 - Internet / reseaux externes

```text
enable
conf t
hostname R1

interface fastethernet0/0
 ip address 192.168.10.254 255.255.255.0
 no shutdown

interface fastethernet0/1
 ip address 192.168.20.254 255.255.255.0
 no shutdown

interface fastethernet1/0
 ip address 192.168.255.1 255.255.255.252
 no shutdown

router rip
 version 2
 no auto-summary
 network 192.168.10.0
 network 192.168.20.0
 network 192.168.255.0

ip route 10.0.2.0 255.255.255.252 192.168.255.2
ip route 10.20.10.0 255.255.255.0 192.168.255.2
ip route 10.20.20.0 255.255.255.0 192.168.255.2

end
wr
```

### 5.2 R2 - Bureaux

```text
enable
conf t
hostname R2

interface fastethernet0/0
 ip address 192.168.255.2 255.255.255.252
 no shutdown

interface fastethernet0/1
 ip address 10.0.1.1 255.255.255.252
 no shutdown

interface fastethernet1/0
 ip address 10.10.10.254 255.255.255.0
 no shutdown

interface fastethernet2/0
 ip address 10.10.20.254 255.255.255.0
 no shutdown

router rip
 version 2
 no auto-summary
 network 192.168.255.0
 network 10.0.0.0

ip route 10.0.2.0 255.255.255.252 10.0.1.2
ip route 10.20.10.0 255.255.255.0 10.0.1.2
ip route 10.20.20.0 255.255.255.0 10.0.1.2

end
wr
```

### 5.3 R3 - Zone serveurs

```text
enable
conf t
hostname R3

interface fastethernet0/0
 ip address 10.0.2.2 255.255.255.252
 no shutdown

interface fastethernet0/1
 ip address 10.20.10.254 255.255.255.0
 no shutdown

interface fastethernet1/0
 ip address 10.20.20.254 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.2.1

end
wr
```

## 6. Configuration du firewall Linux

Le firewall est place entre R2 et R3:

- cote R2: `10.0.1.2/30`;
- cote R3: `10.0.2.1/30`.

La logique de securite est:

- politique par defaut `DROP` sur le trafic forwarde;
- autoriser le trafic de retour `ESTABLISHED,RELATED`;
- autoriser PC-1 IT vers les serveurs;
- bloquer PC-2 HR vers les serveurs;
- bloquer le reseau VPN / Remote Employee vers les serveurs;
- bloquer les serveurs s'ils initient vers les bureaux;
- garder une regle finale `DROP` pour voir les compteurs.

### 6.1 Reset complet du pare-feu et du routage

```bash
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F
route del default 2>/dev/null
```

### 6.2 Activer le routage IPv4

```bash
sysctl -w net.ipv4.ip_forward=1
echo 1 > /proc/sys/net/ipv4/ip_forward
cat /proc/sys/net/ipv4/ip_forward
```

Resultat attendu:

```text
1
```

### 6.3 Ajouter les routes statiques

Retour vers l'office et le VPN via R2:

```bash
route add -net 10.10.0.0 netmask 255.255.0.0 gw 10.0.1.1
route add -net 192.168.10.0 netmask 255.255.255.0 gw 10.0.1.1
```

Aller vers la DMZ / les serveurs via R3:

```bash
route add -net 10.20.0.0 netmask 255.255.0.0 gw 10.0.2.2
route add -net 10.20.10.0 netmask 255.255.255.0 gw 10.0.2.2
route add -net 10.20.20.0 netmask 255.255.255.0 gw 10.0.2.2
```

### 6.4 Appliquer les regles firewall

```bash
iptables -F FORWARD
iptables -P FORWARD ACCEPT

# Politique par defaut: bloquer tout le trafic traverse.
iptables -P FORWARD DROP

# Autoriser les reponses aux connexions deja etablies.
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# PC1 IT autorise vers les reseaux serveurs.
iptables -A FORWARD -s 10.10.10.0/24 -d 10.20.10.0/24 -j ACCEPT
iptables -A FORWARD -s 10.10.10.0/24 -d 10.20.20.0/24 -j ACCEPT

# PC2 HR bloque vers les reseaux serveurs.
iptables -A FORWARD -s 10.10.20.0/24 -d 10.20.10.0/24 -j DROP
iptables -A FORWARD -s 10.10.20.0/24 -d 10.20.20.0/24 -j DROP

# VPN / Remote Employee bloque vers les reseaux serveurs.
iptables -A FORWARD -s 192.168.10.0/24 -d 10.20.10.0/24 -j DROP
iptables -A FORWARD -s 192.168.10.0/24 -d 10.20.20.0/24 -j DROP

# Les serveurs ne peuvent pas initier vers les bureaux.
iptables -A FORWARD -s 10.20.10.0/24 -d 10.10.10.0/24 -j DROP
iptables -A FORWARD -s 10.20.10.0/24 -d 10.10.20.0/24 -j DROP
iptables -A FORWARD -s 10.20.20.0/24 -d 10.10.10.0/24 -j DROP
iptables -A FORWARD -s 10.20.20.0/24 -d 10.10.20.0/24 -j DROP

# Drop final pour visibilite dans les compteurs.
iptables -A FORWARD -j DROP
```

### 6.5 Verifier et sauvegarder les regles

```bash
iptables -L -n -v
iptables-save > /etc/iptables/rules.v4
```

Important: lorsque le projet est reimporte, il faut verifier que les regles firewall et les IP des clients sont toujours presentes. Dans le doute, reappliquer la configuration firewall.

## 7. Configuration du VPN IPsec avec PKI

Le VPN est etabli entre R1 et R2 sur le lien `192.168.255.0/30`.

Trafic protege:

- `192.168.10.0/24` vers `10.10.10.0/24`;
- `192.168.10.0/24` vers `10.10.20.0/24`;
- et le retour inverse depuis R2.

### 7.1 Configuration du CA

```text
enable
conf t
hostname CA

interface fastethernet0/0
 ip address 172.16.1.2 255.255.255.252
 no shutdown

interface fastethernet0/1
 ip address 172.16.2.2 255.255.255.252
 no shutdown

ip http server
end

clock set 10:00:00 Mar 28 2026

conf t
crypto pki server NETSEC-CA
 grant auto
 no shutdown
end
wr
```

Lors du `no shutdown`, le routeur demande une passphrase. Utiliser:

```text
cisco123
```

### 7.2 Configuration PKI et VPN sur R1

```text
enable
conf t
hostname R1

interface fastethernet3/0
 ip address 172.16.1.1 255.255.255.252
 no shutdown
exit

end
clock set 10:05:00 Mar 28 2026

conf t
crypto key generate rsa general-keys label R1KEY modulus 2048

crypto pki trustpoint NETSEC-CA
 enrollment url http://172.16.1.2
 rsakeypair R1KEY
 revocation-check none
exit

crypto pki authenticate NETSEC-CA
crypto pki enroll NETSEC-CA
```

Pendant l'enrollment:

- repondre `yes` a `Request certificate from CA?`;
- repondre `no` aux autres questions si IOS le demande.

Configurer IPsec:

```text
conf t
crypto isakmp policy 1
 encryption aes
 hash sha
 authentication rsa-sig
 group 2
 lifetime 7200
exit

crypto isakmp key 0 Cisco address 192.168.255.2 no-xauth

ip access-list extended VPN-TRAFFIC
 permit ip 192.168.10.0 0.0.0.255 10.10.10.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 10.10.20.0 0.0.0.255
exit

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac

crypto map VPN-MAP 1 ipsec-isakmp
 set peer 192.168.255.2
 set transform-set VPN-SET
 match address VPN-TRAFFIC
exit

interface fastethernet1/0
 crypto map VPN-MAP
exit

crypto isakmp identity hostname
end
wr
```

### 7.3 Configuration PKI et VPN sur R2

```text
enable
conf t
hostname R2

interface fastethernet3/0
 ip address 172.16.2.1 255.255.255.252
 no shutdown
exit

end
clock set 10:06:00 Mar 28 2026

conf t
crypto key generate rsa general-keys label R2KEY modulus 2048

crypto pki trustpoint NETSEC-CA
 enrollment url http://172.16.2.2
 rsakeypair R2KEY
 revocation-check none
exit

crypto pki authenticate NETSEC-CA
crypto pki enroll NETSEC-CA
```

Configurer IPsec:

```text
conf t
crypto isakmp policy 1
 encryption aes
 hash sha
 authentication rsa-sig
 group 2
 lifetime 7200
exit

crypto isakmp key 0 Cisco address 192.168.255.1 no-xauth

ip access-list extended VPN-TRAFFIC
 permit ip 10.10.10.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip 10.10.20.0 0.0.0.255 192.168.10.0 0.0.0.255
exit

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac

crypto map VPN-MAP 1 ipsec-isakmp
 set peer 192.168.255.1
 set transform-set VPN-SET
 match address VPN-TRAFFIC
exit

interface fastethernet0/0
 crypto map VPN-MAP
exit

crypto isakmp identity hostname
end
wr
```

## 8. Tests de connectivite

### 8.1 Verifier les interfaces

Sur les routeurs:

```text
show ip interface brief
show ip route
```

Sur le firewall:

```bash
ip a
ip route
iptables -L -n -v
cat /proc/sys/net/ipv4/ip_forward
```

Sur les VPCS:

```text
show ip
```

### 8.2 Tests depuis PC-1 IT

PC-1 doit pouvoir atteindre les serveurs.

```text
ping 10.20.10.1
ping 10.20.20.1
trace 10.20.20.1
```

Exemple de trace attendue depuis PC-1 vers Server2:

```text
1  10.10.10.254
2  10.0.1.2
3  10.0.2.2
4  10.20.20.1  (ICMP type:3, code:3, Destination port unreachable)
```

Important:

- `trace` utilise UDP, pas ICMP;
- le message `Destination port unreachable` a la fin indique que la destination finale a repondu;
- `trace` montre le chemin;
- `ping` montre seulement si la destination repond.

### 8.3 Tests depuis PC-2 HR

PC-2 doit etre bloque vers les serveurs.

```text
ping 10.20.10.1
ping 10.20.20.1
trace 10.20.10.1
```

Preuve attendue:

- les pings ne passent pas;
- les compteurs `DROP` augmentent sur le firewall.

### 8.4 Tests depuis Remote-Employee

Le Remote-Employee est dans le trafic VPN vers les bureaux, mais il est bloque vers les serveurs par la politique firewall.

```text
ping 10.10.10.1
ping 10.10.20.1
ping 10.20.10.1
ping 10.20.20.1
```

Preuve attendue:

- les tests vers les bureaux peuvent declencher le VPN;
- les tests vers les serveurs sont bloques par le firewall.

### 8.5 Tests depuis Malicious PC

Le Malicious PC ne doit pas avoir d'acces privilegie.

```text
ping 10.10.10.1
ping 10.20.10.1
ping 10.20.20.1
```

Preuve attendue:

- les flux non autorises ne doivent pas traverser le firewall;
- les routes et ACL doivent empecher l'acces aux zones sensibles.

## 9. Verification du VPN

### 9.1 Declencher le tunnel

Depuis Remote-Employee:

```text
ping 10.10.10.1
ping 10.10.20.1
```

### 9.2 Verifier sur R1 et R2

```text
show crypto isakmp sa
show crypto ipsec sa
show crypto pki certificates
show crypto map
```

Preuves attendues:

- `show crypto isakmp sa` montre un etat actif, par exemple `QM_IDLE`;
- `show crypto ipsec sa` montre des paquets encapsules et decapsules;
- les certificats R1/R2 sont presents;
- le crypto map est applique sur la bonne interface.

## 10. Verification du firewall

Avant les tests:

```bash
iptables -L FORWARD -n -v
```

Lancer ensuite les tests depuis PC-1, PC-2 et Remote-Employee, puis relire les compteurs:

```bash
iptables -L FORWARD -n -v
```

Preuves attendues:

- les regles `ACCEPT` de PC-1 vers les serveurs incrementent quand PC-1 teste les serveurs;
- les regles `DROP` de PC-2 vers les serveurs incrementent quand PC-2 teste les serveurs;
- les regles `DROP` du VPN vers les serveurs incrementent quand Remote-Employee teste les serveurs;
- la regle finale `DROP` aide a voir les paquets non prevus.

## 11. Ordre de demonstration simple

### 11.1 Montrer les IP et routes

Sur R1, R2 et R3:

```text
show ip int brief
show ip route
```

Sur le firewall:

```bash
ip a
ip route
iptables -L -n -v
```

### 11.2 Montrer que PC-1 IT accede aux serveurs

Depuis PC-1:

```text
ping 10.20.10.1
ping 10.20.20.1
trace 10.20.20.1
```

### 11.3 Montrer que PC-2 HR est bloque

Depuis PC-2:

```text
ping 10.20.10.1
ping 10.20.20.1
```

Puis sur le firewall:

```bash
iptables -L FORWARD -n -v
```

### 11.4 Montrer que le VPN monte

Depuis Remote-Employee:

```text
ping 10.10.10.1
ping 10.10.20.1
```

Sur R1 et R2:

```text
show crypto isakmp sa
show crypto ipsec sa
```

### 11.5 Montrer que le VPN ne donne pas acces aux serveurs

Depuis Remote-Employee:

```text
ping 10.20.10.1
ping 10.20.20.1
```

Puis sur le firewall:

```bash
iptables -L FORWARD -n -v
```

## 12. Commandes de recuperation utiles

### 12.1 Reappliquer rapidement le firewall

```bash
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F
route del default 2>/dev/null
sysctl -w net.ipv4.ip_forward=1
route add -net 10.10.0.0 netmask 255.255.0.0 gw 10.0.1.1
route add -net 192.168.10.0 netmask 255.255.255.0 gw 10.0.1.1
route add -net 10.20.0.0 netmask 255.255.0.0 gw 10.0.2.2

iptables -F FORWARD
iptables -P FORWARD DROP
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 10.10.10.0/24 -d 10.20.10.0/24 -j ACCEPT
iptables -A FORWARD -s 10.10.10.0/24 -d 10.20.20.0/24 -j ACCEPT
iptables -A FORWARD -s 10.10.20.0/24 -d 10.20.10.0/24 -j DROP
iptables -A FORWARD -s 10.10.20.0/24 -d 10.20.20.0/24 -j DROP
iptables -A FORWARD -s 192.168.10.0/24 -d 10.20.10.0/24 -j DROP
iptables -A FORWARD -s 192.168.10.0/24 -d 10.20.20.0/24 -j DROP
iptables -A FORWARD -s 10.20.10.0/24 -d 10.10.10.0/24 -j DROP
iptables -A FORWARD -s 10.20.10.0/24 -d 10.10.20.0/24 -j DROP
iptables -A FORWARD -s 10.20.20.0/24 -d 10.10.10.0/24 -j DROP
iptables -A FORWARD -s 10.20.20.0/24 -d 10.10.20.0/24 -j DROP
iptables -A FORWARD -j DROP
iptables -L -n -v
```

### 12.2 Reconfigurer rapidement les VPCS

```text
# Remote-Employee
ip 192.168.10.1 255.255.255.0 192.168.10.254
save

# Malicious PC
ip 192.168.20.1 255.255.255.0 192.168.20.254
save

# PC-1 IT
ip 10.10.10.1 255.255.255.0 10.10.10.254
save

# PC-2 HR
ip 10.10.20.1 255.255.255.0 10.10.20.254
save
```

### 12.3 Regler l'horloge avant les certificats

```text
enable
clock set 14:30:00 Mar 30 2026
```

## 13. Resultat attendu final

A la fin du Milestone 1:

- tous les routeurs ont leurs interfaces actives;
- les PC et serveurs ont les bonnes IP et gateways;
- R1, R2, le firewall et R3 routent correctement entre les zones;
- PC-1 IT peut atteindre Server1 et Server2;
- PC-2 HR est bloque vers les serveurs;
- Remote-Employee peut utiliser le VPN vers les bureaux;
- Remote-Employee est bloque vers les serveurs;
- les serveurs ne peuvent pas initier de connexion vers les bureaux;
- le firewall montre les compteurs `ACCEPT` et `DROP`;
- le VPN IPsec montre des SA actives et des compteurs IPsec.

