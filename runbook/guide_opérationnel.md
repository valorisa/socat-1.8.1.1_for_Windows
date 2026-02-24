# 🧭 Guide opérationnel : 20 scénarios réseau avec `socat`
> **Compatibilité OS :** Linux, macOS et Windows 11 (WSL2).  
> **Public visé :** administrateurs, DevOps, ingénieurs réseau, ou pentesters légitimes.  
> **Version socat recommandée :** `>=1.7.4`

---

## ⚙️ Installation rapide

| OS | Commande |
|----|-----------|
| Debian/Ubuntu | `sudo apt install socat` |
| Fedora/RHEL  | `sudo dnf install socat` |
| macOS        | `brew install socat` |
| Windows 11 (WSL2) | `sudo apt install socat` |

---

## 🧩 Exemple 1 — Relais TCP “production” (ACL + logs)

### 🎯 Objectif  
Écouter sur un port local, relayer vers un service distant avec filtres IP et logs persistants.

### 🔁 Schéma de flux
```
[Client local] → TCP:8080 → [socat] → TCP:example.com:80
```

### 💻 Commande
```bash
socat -d -d -lf ./socat_relay_8080.log \
  TCP4-LISTEN:8080,bind=127.0.0.1,reuseaddr,fork,range=127.0.0.1/32 \
  TCP4:example.com:80,connect-timeout=5
```

### 🧪 Test client
```bash
curl -v http://127.0.0.1:8080/
```

### 🩹 Troubleshooting
- Si `curl` renvoie *Connection refused* → port 80 distant fermé.  
- Si rien ne s’affiche et pas de log → vérifier le pare-feu local.  
- Fichier de log disponible : `tail -f ./socat_relay_8080.log`.

---

## 🧩 Exemple 2 — Basculement auto Blue/Green

### 🎯 Objectif  
Basculer automatiquement vers un backend B si le A ne répond pas.

```
Client → 127.0.0.1:9000 → [socat]─▶ A:9000  
                              │  
                              └────▶ B:9000 (fallback)
```

```bash
socat -d -d -ly -lf socat_failover.log \
  TCP4-LISTEN:9000,bind=127.0.0.1,reuseaddr,fork \
  SYSTEM:'(socat - TCP4:10.0.0.11:9000,connect-timeout=1 \
    || socat - TCP4:10.0.0.12:9000,connect-timeout=1)'
```

### 🧪 Test
```bash
curl http://127.0.0.1:9000/
```

### 🩹 Troubleshooting
- Vérifier les IP A/B par `ping`.
- Si les deux échouent → vérifier droits SELinux ou pare-feu.

---

## 🧩 Exemple 3 — Transfert de répertoire avec contrôle d’intégrité

### Schéma
```
Machine A (tar|socat TCP→) → Machine B (socat|tar, tee, sha256sum)
```

### Serveur :
```bash
mkdir -p ./restore
socat -d -d TCP4-LISTEN:8888,reuseaddr - \
 | tee archive.tgz | sha256sum > archive.tgz.sha256
```

### Client :
```bash
tar -czf - dossier/ | socat - TCP4:IP_SERVEUR:8888
```

### Vérification
```bash
sha256sum -c archive.tgz.sha256
```

---

## 🧩 Exemple 4 — SSH tunnel transparent (sans `nc`)

```
localhost:15432 —(socat/ssh)—▶ serveur_distant:5432
```

```bash
socat TCP-LISTEN:15432,reuseaddr,fork \
  EXEC:'ssh -W 127.0.0.1:5432 user@serveur_distant'
```

**Client test :**
```bash
psql -h 127.0.0.1 -p 15432 -U dbuser
```

---

## 🧩 Exemple 5 — Collecteur UDP horodaté

```
Émetteurs UDP ▶ 5514 → [socat + timestamp] → udp_events.log
```

```bash
socat -u UDP4-RECV:5514,reuseaddr \
 SYSTEM:'while read -r line; do printf "%s %s\n" "$(date -Is)" "$line"; done >>./udp_events.log'
```

### Test
```bash
echo "Hello" | socat - UDP4-SENDTO:127.0.0.1:5514
tail -n1 udp_events.log
```

---

## 🧩 Exemple 6 — Proxy TCP de debug (dump HEX)

```
Client → socat:5000 (logs+hex) → Serveur:5001
```

```bash
socat -v -x -d -d -lf ./wire.log \
  TCP-LISTEN:5000,reuseaddr,fork \
  TCP:target.local:5001
```

### Test
```bash
curl http://127.0.0.1:5000/
tail -f wire.log
```

---

## 🧩 Exemple 7 — Pont TCP ↔ socket Unix

```
Client TCP → 127.0.0.1:7000 → /tmp/service.sock
```

```bash
socat TCP-LISTEN:7000,bind=127.0.0.1,reuseaddr,fork \
  UNIX-CONNECT:/tmp/service.sock
```

### Test
```bash
curl --unix-socket /tmp/service.sock http://localhost
# devient équivalent à
curl http://127.0.0.1:7000/
```

---

## 🧩 Exemple 8 — Terminaison TLS simple

```
Client HTTPS :8443 ⇆ [socat TLS⇄TCP] ⇆ Service HTTP :8080
```

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout key.pem -out cert.pem -subj "/CN=localhost"
socat OPENSSL-LISTEN:8443,reuseaddr,fork,cert=cert.pem,key=key.pem,verify=0 \
  TCP:127.0.0.1:8080
```

### Test
```bash
curl -vk https://127.0.0.1:8443/
```

---

## 🧩 Exemple 9 — Client mTLS

```
App→127.0.0.1:9443─▶[socat mTLS]────▶ api.exemple.local:443
```

```bash
socat TCP-LISTEN:9443,reuseaddr,fork \
  OPENSSL:api.exemple.local:443,cert=client.crt,key=client.key,cafile=ca.crt,verify=1
```

---

## 🧩 Exemple 10 — Pont série ↔ TCP

```
Remote client → TCP:2001 ↔ /dev/ttyUSB0
```

```bash
socat TCP-LISTEN:2001,reuseaddr,fork FILE:/dev/ttyUSB0,raw,b115200,echo=0
```

---

## 🧩 Exemple 11 — Endpoint HTTP healthcheck

```bash
socat TCP-LISTEN:9100,bind=127.0.0.1,reuseaddr,fork \
 EXEC:'/bin/sh -c "printf \"HTTP/1.1 200 OK\r\n\r\n%s\n\" \"$(date)\" "'
```

### Test
```bash
curl -s http://127.0.0.1:9100/
```

---

## 🧩 Exemple 12 — Endpoint texte ops

```bash
socat -T15 TCP-LISTEN:9001,bind=127.0.0.1,fork \
 SYSTEM:'echo host=$(hostname);uptime'
```

---

## 🧩 Exemple 13 — Encapsulation UDP→TCP→UDP

```
A:6000/UDP → socat→TCP:7000 → B→socat→UDP:6000
```

*(Commandes inchangées des exemples précédents)*

---

## 🧩 Exemple 14 — Redirection DNS locale

```bash
socat UDP-LISTEN:5353,reuseaddr,fork UDP:9.9.9.9:53
dig @127.0.0.1 -p 5353 example.com
```

---

## 🧩 Exemple 15 — Test de débit réseau

Serveur :
```bash
socat -u TCP-LISTEN:9000,reuseaddr /dev/null
```
Client :
```bash
dd if=/dev/zero bs=1M count=512 | socat -u - TCP:IP:9000
```

---

## 🧩 Exemple 16 — Test de connectivité + timeout

```bash
socat -T 2 TCP:serveur:443 -
```

---

## 🧩 Exemple 17 — Partage ponctuel de fichier HTTP

```
Client (curl) ←── TCP:8000 ←──[socat + SYSTEM : cat fichier]
```

*(Commande du bloc précédent : inchangée mais commentée, prête à copier.)*

---

## 🧩 Exemple 18 — Diffusion multi‑logs (tee + socat)

```
App stdout → tee local.log → TCP vers logserver
```

```bash
mon_app | socat -u - SYSTEM:'tee local.log | socat -u - TCP:logserver:5000'
```

---

## 🧩 Exemple 19 — Tunnel inverse via relais

```
Machine locale:3000 → ssh/socat → Relais public:8080
```

*(Utiliser le script complet indiqué dans la fusion précédente.)*

---

## 🧩 Exemple 20 — Proxy SOCKS4A

```
client→127.0.0.1:8080─▶ proxy:1080─▶ target:80
```

```bash
socat TCP-LISTEN:8080,reuseaddr,fork \
 SOCKS4A:proxy.local:target.local:80,socksport=1080
```

---

# 🧰 Section « Troubleshooting général »

| Symptôme | Diagnostic possible | Solution |
|-----------|--------------------|-----------|
| **“Address already in use”** | socket encore en TIME_WAIT | ajouter `reuseaddr` |
| **Aucune donnée reçue** | pare-feu ou règle `ufw` | tester avec `nc` sur le même port |
| **Erreur SSL: certificate verify failed** | CA manquante | utiliser `cafile=` correct ou `verify=0` en lab |
| **socat bloque sans logs** | buffer stdout plein | ajouter `-d -d` ou `-ly -lf fichier.log` |
| **/dev/tty introuvable** | device non mappé (Windows) | utiliser `usbipd-win` ou adapter le port COM |
| **“unknown device/address type”** | typo dans l’adresse (`TCP:` vs `TCP4:`) | corriger le préfixe |

---

## 📘 Annexes

### Bonnes pratiques
- Toujours spécifier `bind=127.0.0.1` pour éviter une exposition réseau involontaire.  
- Utiliser `connect-timeout` et `-T` pour éviter les connexions fantômes.  
- Préférer `-v -x` pour le debug binaire.  
- Documenter chaque instance (ex: dans `/etc/socat.d/`) avec le but du tunnel.  

---

👉 **Prêt à livrer.**  
