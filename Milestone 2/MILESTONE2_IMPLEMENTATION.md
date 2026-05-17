# Milestone 2 - Reverse Proxy, WAF / Hardening et preuves

Ce document decrit l'implementation complete du Milestone 2 pour le projet 504 Network Security. L'objectif est d'ajouter un reverse proxy Debian/Nginx dans la DMZ, de publier les deux serveurs internes via HTTP/HTTPS, puis de prouver les protections de hardening.

## 1. Objectif du Milestone 2

Le Milestone 2 ajoute une couche de securite devant les serveurs internes:

- un reverse proxy Nginx accessible depuis le reseau externe;
- un load balancing vers Server1 et Server2;
- HTTPS avec certificat autosigne;
- headers de securite;
- limitation des methodes HTTP;
- limitation de taille des requetes;
- rate limiting;
- blocage de fichiers sensibles;
- restriction IP pour `/admin`;
- journalisation des acces et erreurs.

## 2. Adressage utilise

### Backends

| Machine | Interface | Adresse IP | Gateway |
| --- | --- | --- | --- |
| Server1 | eth0 | `10.20.10.1/24` | `10.20.10.254` |
| Server2 | eth0 | `10.20.20.1/24` | `10.20.20.254` |

### Reverse proxy Debian

| Cote | Interface | Adresse IP | Next-hop |
| --- | --- | --- | --- |
| Externe / R1 | eth0 | `10.0.4.2/30` | `10.0.4.1` |
| Interne / R3 | eth1 | `10.0.3.2/30` | `10.0.3.1` |
| NAT temporaire | eth2 | `192.168.122.78/24` | `192.168.122.1` |

### Client externe de test

| Machine | Interface | Adresse IP | Gateway |
| --- | --- | --- | --- |
| Client externe | eth0 | `192.168.30.10/24` | `192.168.30.254` |

## 3. Preparation du reverse proxy

Toutes les commandes Debian sont a executer en root.

### 3.1 Verifier les interfaces

```bash
ip a
ip route
```

Identifier les interfaces du reverse proxy. Dans cette implementation:

- `eth0` est connectee vers R1;
- `eth1` est connectee vers R3;
- `eth2` est l'interface NAT temporaire pour installer les paquets.

### 3.2 Configurer les IP du reverse proxy

```bash
ip addr add 10.0.4.2/30 dev eth0
ip addr add 10.0.3.2/30 dev eth1
ip link set eth0 up
ip link set eth1 up
```

### 3.3 Ajouter les routes vers les serveurs internes

R3 est le next-hop interne du reverse proxy avec l'adresse `10.0.3.1`.

```bash
ip route add 10.20.10.0/24 via 10.0.3.1 dev eth1
ip route add 10.20.20.0/24 via 10.0.3.1 dev eth1
```

### 3.4 Verifier la connectivite interne

```bash
ip route
ping 10.0.3.1
ping 10.20.10.1
ping 10.20.20.1
```

Resultat attendu:

- `10.0.3.1` repond;
- `10.20.10.1` repond;
- `10.20.20.1` repond.

## 4. Internet temporaire sur le reverse proxy

L'interface NAT est utilisee uniquement pour installer Nginx et OpenSSL.

### 4.1 Configurer l'interface NAT

```bash
ip addr add 192.168.122.78/24 dev eth2
ip link set eth2 up
```

### 4.2 Configurer le DNS

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

### 4.3 Mettre la route par defaut temporaire vers NAT

Si une route par defaut existe deja via R3, la supprimer avant d'ajouter la route NAT:

```bash
ip route del default via 10.0.3.1 dev eth1
ip route add default via 192.168.122.1 dev eth2
```

Si la suppression retourne une erreur parce que la route n'existe pas, continuer avec l'ajout de la route NAT.

### 4.4 Verifier l'acces Internet

```bash
ip route
ping 192.168.122.1
ping 8.8.8.8
```

## 5. Installation Nginx et OpenSSL sur le reverse proxy

```bash
apt update
apt install -y nginx openssl
```

Creer les fichiers de logs si necessaire:

```bash
mkdir -p /var/log/nginx
touch /var/log/nginx/access.log
touch /var/log/nginx/error.log
```

Sauvegarder la configuration Nginx d'origine:

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

Creer le certificat autosigne:

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/reverseproxy.key \
  -out /etc/nginx/ssl/reverseproxy.crt \
  -subj "/C=BE/ST=Brussels/L=Brussels/O=ULB/CN=10.0.4.2"
```

## 6. Configuration Nginx durcie du reverse proxy

Remplacer le contenu de `/etc/nginx/nginx.conf` par la configuration suivante:

```bash
tee /etc/nginx/nginx.conf > /dev/null << 'EOF'
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    server_tokens off;

    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=5r/s;

    upstream backend_servers {
        server 10.20.10.1;
        server 10.20.20.1;
    }

    server {
        listen 80;
        server_name _;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Content-Security-Policy "default-src 'self'" always;
        add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

        client_max_body_size 1m;

        limit_req zone=req_limit burst=10 nodelay;

        location / {
            limit_except GET HEAD POST { deny all; }

            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ /\.(ht|git|env) {
            deny all;
        }

        location ~* /(nginx\.conf|passwd|shadow) {
            deny all;
        }

        location = /admin {
            allow 10.0.4.1;
            deny all;
        }
    }

    server {
        listen 443 ssl;
        server_name _;

        ssl_certificate /etc/nginx/ssl/reverseproxy.crt;
        ssl_certificate_key /etc/nginx/ssl/reverseproxy.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_session_timeout 10m;
        ssl_session_cache shared:SSL:10m;

        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Content-Security-Policy "default-src 'self'" always;
        add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

        client_max_body_size 1m;

        limit_req zone=req_limit burst=10 nodelay;

        location / {
            limit_except GET HEAD POST { deny all; }

            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ /\.(ht|git|env) {
            deny all;
        }

        location ~* /(nginx\.conf|passwd|shadow) {
            deny all;
        }

        location = /admin {
            allow 10.0.4.1;
            deny all;
        }
    }
}
EOF
```

Tester puis relancer Nginx:

```bash
nginx -t
pkill nginx
nginx
ss -ltnp | grep ':80\|:443'
```

Resultat attendu:

- `nginx -t` affiche que la configuration est valide;
- Nginx ecoute sur `0.0.0.0:80`;
- Nginx ecoute sur `0.0.0.0:443`.

## 7. Remettre la route normale du reverse proxy

Apres l'installation des paquets, supprimer la route NAT temporaire:

```bash
ip route del default via 192.168.122.1 dev eth2
ip route add default via 10.0.3.1 dev eth1
ip route
```

Conserver ou remettre les routes specifiques vers les backends:

```bash
ip route add 10.20.10.0/24 via 10.0.3.1 dev eth1
ip route add 10.20.20.0/24 via 10.0.3.1 dev eth1
```

Si ces routes existent deja, le systeme peut afficher `RTNETLINK answers: File exists`. Ce n'est pas bloquant.

## 8. Installation et configuration des serveurs backend

Les serveurs backend doivent servir une page locale sur le port 80. L'interface NAT `eth1` est utilisee temporairement pour installer Nginx.

### 8.1 Server1

Mettre la route NAT temporaire:

```bash
ip route del default via 10.20.10.254 dev eth0
ip route add default via 192.168.122.1 dev eth1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Installer et configurer Nginx:

```bash
apt update
apt install -y nginx
nginx
echo '<html><body><h1>Server1</h1></body></html>' > /var/www/html/index.html
ss -ltnp | grep ':80'
curl http://127.0.0.1/
```

Remettre la route normale:

```bash
ip route del default via 192.168.122.1 dev eth1
ip route add default via 10.20.10.254 dev eth0
ip route
```

### 8.2 Server2

Mettre la route NAT temporaire:

```bash
ip route del default via 10.20.20.254 dev eth0
ip route add default via 192.168.122.1 dev eth1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Installer et configurer Nginx:

```bash
apt update
apt install -y nginx
nginx
echo '<html><body><h1>Server2</h1></body></html>' > /var/www/html/index.html
ss -ltnp | grep ':80'
curl http://127.0.0.1/
```

Remettre la route normale:

```bash
ip route del default via 192.168.122.1 dev eth1
ip route add default via 10.20.20.254 dev eth0
ip route
```

## 9. Configuration du nouveau client externe

### 9.1 Configuration R1

Sur R1:

```cisco
enable
conf t
interface fastethernet2/0
 ip address 192.168.30.254 255.255.255.0
 no shutdown
end
wr
```

### 9.2 Configuration du client Linux externe

```bash
ip addr add 192.168.30.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.30.254
ip route
ping 192.168.30.254
ping 10.0.4.2
```

### 9.3 Route de retour sur le reverse proxy

Sur le reverse proxy:

```bash
ip route add 192.168.30.0/24 via 10.0.4.1 dev eth0
ip route
```

## 10. Tests fonctionnels

### 10.1 Tester les backends depuis le reverse proxy

```bash
curl http://10.20.10.1
curl http://10.20.20.1
```

Resultats attendus:

```html
<html><body><h1>Server1</h1></body></html>
<html><body><h1>Server2</h1></body></html>
```

### 10.2 Tester le proxy localement

```bash
curl http://127.0.0.1
curl -k https://127.0.0.1
```

Resultat attendu: Nginx renvoie alternativement `Server1` ou `Server2` selon le backend choisi par l'upstream.

### 10.3 Tester le proxy via son IP externe

```bash
curl http://10.0.4.2
curl -k https://10.0.4.2
curl -I http://10.0.4.2
curl -k -I https://10.0.4.2
```

### 10.4 Tester depuis le client externe

```bash
ping 10.0.4.2
curl http://10.0.4.2
curl -k https://10.0.4.2
```

## 11. Preuves de hardening avant / apres

Cette section donne les commandes a executer pendant la demonstration et les preuves attendues.

### 11.1 Masquage de la version serveur

```bash
curl -I http://10.0.4.2
curl -k -I https://10.0.4.2
```

Preuve attendue:

- le header `Server` ne doit pas afficher la version exacte de Nginx;
- `server_tokens off;` empeche l'affichage du numero de version.

### 11.2 Headers de securite

```bash
curl -I http://10.0.4.2
curl -k -I https://10.0.4.2
```

Headers attendus:

```text
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'
Permissions-Policy: geolocation=(), microphone=(), camera=()
Strict-Transport-Security: max-age=31536000
```

Note: `Strict-Transport-Security` apparait sur HTTPS, pas necessairement sur HTTP.

### 11.3 TLS / HTTPS

```bash
ss -ltnp | grep ':443'
curl -k -I https://10.0.4.2
openssl s_client -connect 10.0.4.2:443
```

Preuve attendue:

- le port `443` est en ecoute;
- `curl -k` obtient une reponse HTTP;
- `openssl s_client` montre un certificat avec `CN=10.0.4.2`;
- TLSv1.2 ou TLSv1.3 est utilise.

### 11.4 Methodes HTTP interdites

```bash
curl -i -X DELETE http://10.0.4.2
curl -k -i -X DELETE https://10.0.4.2
```

Preuve attendue:

- la methode `DELETE` est refusee;
- le code attendu est `403 Forbidden`.

### 11.5 Limite de taille des requetes

Creer un fichier de 2 MB:

```bash
dd if=/dev/zero of=/tmp/bigfile bs=1M count=2
```

Tester l'envoi HTTP et HTTPS:

```bash
curl -i -X POST --data-binary @/tmp/bigfile http://10.0.4.2
curl -k -i -X POST --data-binary @/tmp/bigfile https://10.0.4.2
```

Preuve attendue:

- la requete est refusee car `client_max_body_size` est fixe a `1m`;
- le code attendu est `413 Request Entity Too Large`.

### 11.6 Rate limiting

```bash
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" http://10.0.4.2; done
for i in $(seq 1 30); do curl -k -s -o /dev/null -w "%{http_code}\n" https://10.0.4.2; done
```

Preuve attendue:

- certaines requetes retournent `200`;
- quand la limite est depassee, certaines requetes retournent `503`;
- cela prouve l'application de `limit_req`.

### 11.7 Blocage de fichiers sensibles

```bash
curl -i http://10.0.4.2/.git
curl -i http://10.0.4.2/.env
curl -i http://10.0.4.2/nginx.conf

curl -k -i https://10.0.4.2/.git
curl -k -i https://10.0.4.2/.env
curl -k -i https://10.0.4.2/nginx.conf
```

Preuve attendue:

- les chemins sensibles sont refuses;
- le code attendu est `403 Forbidden`.

### 11.8 Logging

Dans un terminal sur le reverse proxy:

```bash
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

Dans un autre terminal:

```bash
curl http://10.0.4.2
curl -X DELETE http://10.0.4.2
```

Preuve attendue:

- la requete normale apparait dans `access.log`;
- la requete interdite apparait aussi avec son code HTTP;
- les erreurs de securite ou de proxy peuvent apparaitre dans `error.log`.

### 11.9 Restriction IP sur `/admin`

```bash
curl -i http://10.0.4.2/admin
curl -k -i https://10.0.4.2/admin
```

Preuve attendue:

- seuls les clients autorises par `allow 10.0.4.1;` peuvent acceder a `/admin`;
- depuis un client non autorise, le code attendu est `403 Forbidden`.

## 12. Verifications finales

### 12.1 Sur le reverse proxy

```bash
ip a
ip route
nginx -t
ss -ltnp | grep ':80\|:443'
tail -n 20 /var/log/nginx/access.log
tail -n 20 /var/log/nginx/error.log
```

Points a verifier:

- `eth0` possede `10.0.4.2/30`;
- `eth1` possede `10.0.3.2/30`;
- les routes vers `10.20.10.0/24`, `10.20.20.0/24` et `192.168.30.0/24` existent;
- Nginx ecoute sur `80` et `443`;
- la configuration Nginx est valide.

### 12.2 Sur R1

```text
show ip int brief
show ip route
```

Points a verifier:

- l'interface vers le client externe est active;
- le reseau `192.168.30.0/24` est present;
- la route vers le reverse proxy est correcte.

### 12.3 Sur Server1 et Server2

```bash
ss -ltnp | grep ':80'
curl http://127.0.0.1
ip route
```

Points a verifier:

- Nginx ecoute sur le port `80`;
- Server1 affiche `Server1`;
- Server2 affiche `Server2`;
- la route par defaut est revenue vers le routeur interne normal.

## 13. Ordre de demonstration simple

Utiliser cet ordre pendant la presentation pour montrer rapidement que tout fonctionne.

### 13.1 Montrer les backends

```bash
curl http://10.20.10.1
curl http://10.20.20.1
```

### 13.2 Montrer le reverse proxy HTTP et HTTPS

```bash
curl http://10.0.4.2
curl -k https://10.0.4.2
```

### 13.3 Montrer les headers

```bash
curl -I http://10.0.4.2
curl -k -I https://10.0.4.2
```

### 13.4 Montrer le blocage des methodes

```bash
curl -i -X DELETE http://10.0.4.2
```

### 13.5 Montrer la limite de taille

```bash
dd if=/dev/zero of=/tmp/bigfile bs=1M count=2
curl -i -X POST --data-binary @/tmp/bigfile http://10.0.4.2
```

### 13.6 Montrer le rate limiting

```bash
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" http://10.0.4.2; done
```

### 13.7 Montrer le blocage des fichiers sensibles

```bash
curl -i http://10.0.4.2/.git
curl -i http://10.0.4.2/.env
curl -i http://10.0.4.2/nginx.conf
```

### 13.8 Montrer la restriction `/admin`

```bash
curl -i http://10.0.4.2/admin
```

### 13.9 Montrer les logs

```bash
tail -n 20 /var/log/nginx/access.log
tail -n 20 /var/log/nginx/error.log
```

## 14. Commandes de recuperation utiles

### Relancer Nginx proprement

```bash
nginx -t
pkill nginx
nginx
ss -ltnp | grep ':80\|:443'
```

### Verifier rapidement le routage

```bash
ip a
ip route
ping 10.0.4.1
ping 10.0.3.1
ping 10.20.10.1
ping 10.20.20.1
```

### Voir les erreurs Nginx

```bash
tail -n 50 /var/log/nginx/error.log
```

## 15. Resultat attendu final

A la fin du Milestone 2:

- Server1 et Server2 repondent localement sur HTTP;
- le reverse proxy atteint Server1 et Server2;
- le client externe atteint le reverse proxy sur `10.0.4.2`;
- HTTP fonctionne sur le port `80`;
- HTTPS fonctionne sur le port `443`;
- les headers de securite sont visibles;
- les methodes dangereuses sont bloquees;
- les gros uploads sont refuses;
- le rate limiting produit des refus quand trop de requetes sont envoyees;
- les chemins sensibles sont bloques;
- `/admin` est restreint par IP;
- les logs Nginx prouvent les acces et les refus.

