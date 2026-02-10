# Configuration VPN Site-to-Site

![Schéma VPN Site-to-Site](Capture%20d'écran%202026-02-10%20153303.png)

---

## 1️⃣ Compréhension de l'architecture

### 🔹 Ce qui chiffre le trafic

👉 **QG-R1-CORE ↔ BR-R1-CORE**

Le VPN **NE PASSE PAS** par :
- QG-R1-DIST
- BR-R1-DIST
- Les switches
- Le cœur FAI (R1 / R2 en OSPF) = simple transport IP

### 🔹 Réseaux à chiffrer (trafic intéressant)

| Site | Réseau LAN |
|------|------------|
| QG   | 192.168.10.0/24 |
| BR   | 192.168.20.0/24 |

### 🔹 Réseaux WAN

- **QG-R1-CORE ↔ FAI** : 10.0.0.0/30
- **BR-R1-CORE ↔ FAI** : 11.0.0.0/30
- **Backbone FAI** : 12.0.0.0 (OSPF)

> 📌 Le VPN ignore complètement OSPF

---

## 2️⃣ Étape 1 — ACL (trafic intéressant)

### 📍 Sur QG-R1-CORE

```cisco
access-list VPN-ACL permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```

### 📍 Sur BR-R1-CORE (inverse !)

```cisco
access-list VPN-ACL permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

> ⚠️ **Erreur classique** : oublier l'inversion → VPN KO.

---

## 3️⃣ Étape 2 — IKE Phase 1 (ISAKMP / IKEv1)

👉 **STRICTEMENT IDENTIQUE des deux côtés**

```cisco
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
```

### 🔑 Clé pré-partagée

**QG-R1-CORE**
```cisco
crypto isakmp key VPN123 address 11.0.0.1
```

**BR-R1-CORE**
```cisco
crypto isakmp key VPN123 address 10.0.0.1
```

---

## 4️⃣ Étape 3 — IKE Phase 2 (IPsec)

### 🔐 Transform-set

```cisco
crypto ipsec transform-set ESP-AES-SHA esp-aes esp-sha-hmac
```

---

## 5️⃣ Étape 4 — Crypto Map

### 📍 QG-R1-CORE

```cisco
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 11.0.0.1
 set transform-set ESP-AES-SHA
 match address VPN-ACL
```

### 📍 BR-R1-CORE

```cisco
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 10.0.0.1
 set transform-set ESP-AES-SHA
 match address VPN-ACL
```

---

## 6️⃣ Étape 5 — Appliquer la crypto map (CRITIQUE)

**Sur l'interface WAN UNIQUEMENT**

### QG-R1-CORE

```cisco
interface GigabitEthernet0/0
 crypto map VPN-MAP
```

### BR-R1-CORE

```cisco
interface GigabitEthernet0/0
 crypto map VPN-MAP
```

> 📌 Si tu l'appliques sur la mauvaise interface → rien ne marche

---

## 7️⃣ Étape 6 — Routage (souvent oublié)

Les routeurs doivent savoir joindre les LAN distants (avant chiffrement).

### QG-R1-CORE

```cisco
ip route 192.168.20.0 255.255.255.0 10.0.0.2
```

### BR-R1-CORE

```cisco
ip route 192.168.10.0 255.255.255.0 11.0.0.2
```

> OSPF reste uniquement chez le FAI 👍

---

## 8️⃣ Étape 7 — Tests

### Depuis un PC QG :

```bash
ping 192.168.20.10
```

### Vérifications VPN :

```cisco
show crypto isakmp sa
show crypto ipsec sa
```

**Tu dois voir :**
- `QM_IDLE`
- `encaps / decaps` qui augmentent

---

## 9️⃣ Résumé rapide (vision pro)

✔ Architecture très bien pensée  
✔ VPN routeur ↔ routeur (plus simple que ASA)  
✔ OSPF hors VPN  
✔ **Succès** =
- ACL correcte
- Phase 1 identique
- Phase 2 identique
- Crypto map sur la bonne interface

---

## 👉 Prochaines étapes possibles

- Ajouter plusieurs LAN dans le VPN
- Mettre du PFS
- Migrer vers IKEv2
- Faire un schéma de dépannage (quoi vérifier quand ça ne monte pas)
