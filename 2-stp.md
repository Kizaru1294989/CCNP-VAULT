# ENCOR 350-401 — Chapitre 2 : Spanning Tree Protocol

> Cours de révision complet : 802.1D, RSTP, variantes, vérification, pannes, scénarios et exercices corrigés.
> Les chapitres 3 (tuning : PortFast, BPDU Guard, Root Guard…) et 4 (MST) sont résumés en annexe.

---

## Sommaire

1. [Pourquoi STP existe](#1-pourquoi-stp-existe)
2. [Vocabulaire de base](#2-vocabulaire-de-base)
3. [Le bridge ID](#3-le-bridge-id)
4. [Le coût STP](#4-le-coût-stp)
5. [Construire la topologie en 3 étapes](#5-construire-la-topologie-en-3-étapes)
6. [Scénario fil rouge : 5 switches calculés pas à pas](#6-scénario-fil-rouge--5-switches-calculés-pas-à-pas)
7. [États et types de ports 802.1D](#7-états-et-types-de-ports-8021d)
8. [Timers](#8-timers)
9. [Vérification : lire les sorties](#9-vérification--lire-les-sorties)
10. [Changements de topologie (TCN)](#10-changements-de-topologie-tcn)
11. [Pannes directes et indirectes](#11-pannes-directes-et-indirectes)
12. [RSTP (802.1w)](#12-rstp-8021w)
13. [Les variantes : STP, PVST+, RSTP, Rapid PVST+, MST](#13-les-variantes--stp-pvst-rstp-rapid-pvst-mst)
14. [Configuration essentielle](#14-configuration-essentielle)
15. [Exercices corrigés](#15-exercices-corrigés)
16. [Pièges d'examen](#16-pièges-dexamen)
17. [Aide-mémoire](#17-aide-mémoire)
18. [Annexe : aperçu des chapitres 3 et 4](#18-annexe--aperçu-des-chapitres-3-et-4)

---

## 1. Pourquoi STP existe

On met des liens redondants entre switches pour la haute disponibilité. Problème : **en L2, la redondance crée des boucles**, et Ethernet n'a **aucun mécanisme anti-boucle** (pas de TTL dans une trame).

Conséquences d'une boucle L2 :

| Problème | Ce qui se passe |
|---|---|
| **Broadcast storm** | un broadcast tourne à l'infini et se multiplie à chaque switch, jusqu'à saturer les liens et les CPU |
| **Instabilité de la table MAC (MAC flapping)** | la même MAC est apprise alternativement sur plusieurs ports |
| **Trames dupliquées** | l'hôte destinataire reçoit plusieurs copies de la même trame |

**La solution STP** : bloquer logiquement certains ports pour obtenir un **arbre** (pas de boucle), tout en gardant les liens bloqués **en réserve** si un lien actif tombe.

> L'idée : garder la redondance **physique**, supprimer la redondance **logique**.

---

## 2. Vocabulaire de base

| Terme | Définition |
|---|---|
| **Root bridge** | le switch au centre de l'arbre. Tous ses ports actifs sont designated (forwarding). |
| **BPDU** (Bridge Protocol Data Unit) | message STP échangé entre switches (multicast `0180.c200.0000`). |
| **Configuration BPDU** | annonce l'identité du root, le coût, les timers. En 802.1D, **générée uniquement par le root** puis relayée. |
| **TCN BPDU** (Topology Change Notification) | signale un changement de topologie, envoyée **vers le root**. |
| **Root path cost** | coût total d'un switch vers le root (somme des coûts des ports **d'entrée** sur le chemin). |
| **System priority** | priorité configurable du switch (32768 par défaut). |
| **System ID extension** | le VLAN ID, ajouté à la priorité (PVST+). |
| **Root port (RP)** | sur chaque switch non-root, **le** port qui mène au root par le meilleur chemin. Un seul par switch. |
| **Designated port (DP)** | sur chaque **segment**, le port qui forwarde vers ce segment. Un seul par segment. |
| **Blocking port** | tout port ni root ni designated : il ne forwarde pas. |

> Règles d'or :
> - **1 root bridge** par instance STP (donc par VLAN en PVST+)
> - **1 root port** par switch non-root
> - **1 designated port** par segment
> - tout le reste est **bloqué**

---

## 3. Le bridge ID

Le bridge ID (BID) fait **8 octets** :

```
| Priorité (4 bits) | System ID extension = VLAN (12 bits) | MAC (48 bits) |
|<------------- 2 octets ------------------------------>|<-- 6 octets ->|
```

- Comme 12 bits sont pris par le VLAN, il ne reste que **4 bits** de priorité : on ne peut configurer que des **multiples de 4096**, de **0 à 61440**.
- La priorité **affichée** = priorité configurée **+ VLAN ID**.

| Configuré | VLAN | Affiché |
|---|---|---|
| 32768 (défaut) | 1 | **32769** |
| 32768 | 10 | **32778** |
| 24576 | 20 | **24596** |
| 4096 | 99 | **4195** |

**Comparaison de deux BID** : la plus basse valeur gagne. On compare d'abord la **priorité**, puis la **MAC** en cas d'égalité.

> ⚠️ `spanning-tree vlan 10 priority 1000` → **refusé** (pas un multiple de 4096).

---

## 4. Le coût STP

### 4.1 Tableau des coûts

| Vitesse | Short (par cœur) | Long |
|---|---|---|
| 10 Mbps | **100** | 2 000 000 |
| 100 Mbps | **19** | 200 000 |
| 1 Gbps | **4** | 20 000 |
| 10 Gbps | **2** | 2 000 |
| 20 Gbps | 1 | 1 000 |
| 100 Gbps | 1 | 200 |
| 1 Tbps | 1 | 20 |
| 10 Tbps | 1 | 2 |

- Mnémo short : **100 → 19 → 4 → 2 → 1**
- Formule long : **20 000 000 / vitesse en Mbps**
- Le mode short ne distingue plus rien au-delà de 20 Gbps (tout vaut 1), d'où le mode long.
- Commande : `spanning-tree pathcost method {short | long}`

### 4.2 Comment se calcule le root path cost

Chaque BPDU transporte le **coût annoncé** par l'émetteur. Le récepteur **ajoute le coût de son port d'entrée**.

```
ROOT ──(1G)──> SW2 ──(1G)──> SW4 ──(100M)──> SW6

Root annonce 0
SW2 : 0 + 4  = 4    → annonce 4
SW4 : 4 + 4  = 8    → annonce 8
SW6 : 8 + 19 = 27
```

> Le root annonce toujours un coût de **0**.

---

## 5. Construire la topologie en 3 étapes

### Étape 1 — Élire le root bridge

Au démarrage, **chaque switch se croit root** et envoie ses BPDU. Dès qu'il reçoit une BPDU **supérieure** (BID plus bas), il arrête de s'annoncer et relaie celle du meilleur.

**Gagnant : le plus petit bridge ID** (priorité, puis MAC).

> Avec les priorités par défaut, c'est le switch à la **MAC la plus basse** qui gagne, souvent le plus vieux du réseau. C'est pour ça qu'on fixe toujours le root à la main.

### Étape 2 — Chaque switch non-root choisit son root port

Critères dans l'ordre, chacun ne sert qu'en cas d'égalité au précédent :

| # | Critère | Le plus bas gagne |
|---|---|---|
| 1 | **Root path cost** | coût total vers le root |
| 2 | **Priorité système de l'émetteur** | switch voisin qui envoie la BPDU |
| 3 | **MAC de l'émetteur** | switch voisin |
| 4 | **Priorité de port de l'émetteur** | 128 par défaut |
| 5 | **Numéro de port de l'émetteur** | |

> ⚠️ **Piège n°1** : à partir du critère 2, ce sont **toujours les valeurs de l'émetteur** (le voisin en amont), jamais celles du switch local.

Mnémo : **coût → qui m'envoie → par quel port il m'envoie**.

### Étape 3 — Élire un designated port par segment

Sur chaque segment (lien), un seul des deux ports forwarde :

| # | Critère |
|---|---|
| 1 | **Root path cost** le plus bas du switch |
| 2 | **Bridge ID** le plus bas (priorité, puis MAC) |
| 3 | **Port ID** le plus bas (priorité de port, puis numéro) |

- Sur un segment relié au root, le port du root est **toujours** designated (coût 0).
- Le port perdant devient **blocking**.

> ⚠️ Le coût passe **avant** la priorité : un switch en priorité 4096 avec un coût de 8 perd contre un switch en priorité 32768 avec un coût de 4.

---

## 6. Scénario fil rouge : 5 switches calculés pas à pas

### 6.1 Topologie (celle du Cert Guide, Figure 2-2)

Tous les liens sont en **1 Gbps** (coût 4), toutes les priorités par défaut (VLAN 1 → 32769).

| Switch | MAC |
|---|---|
| SW1 | 0062.ec9d.c500 |
| SW2 | 0081.c4ff.8b00 |
| SW3 | 189c.5d11.9980 |
| SW4 | 70df.2f22.b880 |
| SW5 | bc67.1c5c.9300 |

```
                     SW1
              Gi1/0/2   Gi1/0/3
                /           \
         Gi1/0/1             Gi1/0/1
           SW2 Gi1/0/3 ─── Gi1/0/2 SW3
         Gi1/0/4                 Gi1/0/5
           |                        |
         Gi1/0/2                 Gi1/0/3
           SW4 Gi1/0/5 ─── Gi1/0/4 SW5
               Gi1/0/6 ─── Gi1/0/5
```

### 6.2 Étape 1 : root bridge

Toutes les priorités sont égales (32769) → la MAC la plus basse gagne : **SW1** (`0062…`).

### 6.3 Étape 2 : root ports

| Switch | Chemins possibles | Coût | Root port |
|---|---|---|---|
| SW2 | direct vers SW1 | 0 + 4 = **4** | **Gi1/0/1** |
| | via SW3 | 4 + 4 = 8 | |
| SW3 | direct vers SW1 | **4** | **Gi1/0/1** |
| | via SW2 | 8 | |
| SW4 | via SW2 (annonce 4) | 4 + 4 = **8** | **Gi1/0/2** |
| | via SW5 (annonce 8) | 8 + 4 = 12 | |
| SW5 | via SW3 (annonce 4) | 4 + 4 = **8** | **Gi1/0/3** |
| | via SW4 (annonce 8) | 12 | |

Tous les root ports se décident ici au **coût** (critère 1).

### 6.4 Étape 3 : designated ports

| Segment | Candidats | Décision | Designated | Bloqué |
|---|---|---|---|---|
| SW1 – SW2 | SW1 coût 0 / SW2 coût 4 | coût | SW1 Gi1/0/2 | — (SW2 Gi1/0/1 est RP) |
| SW1 – SW3 | SW1 coût 0 / SW3 coût 4 | coût | SW1 Gi1/0/3 | — (SW3 Gi1/0/1 est RP) |
| SW2 – SW3 | coût 4 / coût 4 | **égalité** → BID : SW2 `0081…` < SW3 `189c…` | **SW2 Gi1/0/3** | **SW3 Gi1/0/2** |
| SW2 – SW4 | SW2 coût 4 / SW4 coût 8 | coût | SW2 Gi1/0/4 | — (SW4 Gi1/0/2 est RP) |
| SW3 – SW5 | SW3 coût 4 / SW5 coût 8 | coût | SW3 Gi1/0/5 | — (SW5 Gi1/0/3 est RP) |
| SW4 – SW5 (lien 1) | coût 8 / coût 8 | **égalité** → BID : SW4 `70df…` < SW5 `bc67…` | **SW4 Gi1/0/5** | **SW5 Gi1/0/4** |
| SW4 – SW5 (lien 2) | idem | idem | **SW4 Gi1/0/6** | **SW5 Gi1/0/5** |

### 6.5 Résultat final

```
                     SW1 (ROOT)
               DP Gi1/0/2   Gi1/0/3 DP
                /                 \
         RP Gi1/0/1             Gi1/0/1 RP
           SW2 DP Gi1/0/3 ─X─ Gi1/0/2 BLK SW3
         DP Gi1/0/4                 Gi1/0/5 DP
           |                           |
         RP Gi1/0/2                 Gi1/0/3 RP
           SW4 DP Gi1/0/5 ─X─ Gi1/0/4 BLK SW5
               DP Gi1/0/6 ─X─ Gi1/0/5 BLK
```

**3 ports bloqués** : SW3 Gi1/0/2, SW5 Gi1/0/4, SW5 Gi1/0/5. L'arbre est sans boucle.

### 6.6 Variante : et si on mettait SW5 en priorité 4096 ?

- SW5 a désormais le plus petit BID → **SW5 devient root**.
- Tout se recalcule autour de lui : ses ports deviennent tous designated, SW3 et SW4 prennent leur root port vers SW5, etc.

> Changer une priorité ne « déplace » pas un port bloqué : ça reconstruit **tout l'arbre**.

---

## 7. États et types de ports 802.1D

### 7.1 Les 5 états

| État | Reçoit les BPDU | Envoie les BPDU | Apprend les MAC | Forwarde | Durée |
|---|---|---|---|---|---|
| **Disabled** | non | non | non | non | port shutdown |
| **Blocking** | **oui** | non | non | non | jusqu'à Max Age |
| **Listening** | oui | **oui** | non | non | **15 s** (Forward Delay) |
| **Learning** | oui | oui | **oui** | non | **15 s** (Forward Delay) |
| **Forwarding** | oui | oui | oui | **oui** | — |

Ordre : **Blocking → Listening → Learning → Forwarding**.

- **Listening** : le port participe à l'élection, mais n'apprend rien (sinon il apprendrait des MAC d'une topologie pas encore stable).
- **Learning** : il remplit la table MAC **avant** de forwarder, pour éviter un flooding massif à l'ouverture.

### 7.2 Les 3 types de ports 802.1D

| Type | Rôle |
|---|---|
| **Root port** | meilleur chemin vers le root, forwarding |
| **Designated port** | forwarde vers son segment |
| **Blocking port** | ne forwarde pas |

> Alternate / backup sont des rôles **RSTP**. Edge / non-edge / point-to-point sont des **types** RSTP.

---

## 8. Timers

| Timer | Défaut | Rôle |
|---|---|---|
| **Hello** | **2 s** | intervalle d'envoi des configuration BPDU par le root |
| **Max Age** | **20 s** | durée de validité d'une info reçue sans rafraîchissement |
| **Forward Delay** | **15 s** | durée de l'état listening, puis de l'état learning |

Calculs à connaître :

| Situation | Calcul | Total |
|---|---|---|
| Port qui passe de blocking à forwarding (panne **directe**) | 15 + 15 | **30 s** |
| Panne **indirecte** | 20 + 15 + 15 | **50 s** |
| Durée du flag TC envoyé par le root | 20 + 15 | **35 s** |
| Aging MAC pendant un changement de topologie | Forward Delay | **15 s** (au lieu de 300 s) |

> Les timers sont **imposés par le root** : ceux annoncés dans ses BPDU s'appliquent à tous. On ne les modifie donc que sur le root (et on évite de les toucher).

---

## 9. Vérification : lire les sorties

### 9.1 `show spanning-tree root`

```
SW3# show spanning-tree root
                                        Root    Hello Max Fwd
Vlan                   Root ID          Cost    Time  Age Dly  Root Port
---------------- -------------------- --------- ----- --- ---  ------------
VLAN0001         32769 0062.ec9d.c500         4    2   20  15  Gi1/0/1
VLAN0010         32778 0062.ec9d.c500         4    2   20  15  Gi1/0/1
```

- Une ligne par VLAN.
- **Root Port renseigné** → ce switch **n'est pas** root.
- **Root Cost** = coût total de **ce switch** vers le root.
- Sur le root lui-même : Root Cost = **0**, Root Port **vide**.

### 9.2 `show spanning-tree vlan 1`

```
SW3# show spanning-tree vlan 1

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0062.ec9d.c500
             Cost        4
             Port        1 (GigabitEthernet1/0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     189c.5d11.9980
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1             Root FWD 4         128.1    P2p
Gi1/0/2             Altn BLK 4         128.2    P2p
Gi1/0/5             Desg FWD 4         128.5    P2p
```

Comment la lire :

| Ligne | Ce qu'elle dit |
|---|---|
| `protocol ieee` | 802.1D → **PVST+**. (`rstp` = Rapid PVST+, `mstp` = MST) |
| Root ID vs Bridge ID | **adresses différentes** → ce switch n'est pas root. Si elles sont identiques : « This bridge is the root ». |
| `Cost 4` / `Port 1` | coût vers le root et root port |
| `priority 32768 sys-id-ext 1` | priorité configurée + VLAN |
| `Role` | Root / Desg / Altn / Back |
| `Sts` | FWD / BLK / LIS / LRN |
| `Cost` | coût **du port** (pas le coût total) |
| `Prio.Nbr` | port ID = priorité de port . numéro |
| `Type` | P2p, Shr (shared), Edge |

> Cisco affiche **Altn** même en PVST+ : c'est un affichage, le port est bien un blocking port 802.1D.

### 9.3 `show interfaces trunk` (dernière section)

```
SW3# show interfaces trunk
...
Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/1     1,10,20,99
Gi1/0/2     none
Gi1/0/5     1,10,20,99
```

→ Gi1/0/2 est **bloqué par STP pour tous les VLAN**. C'est la façon la plus rapide de voir les VLAN réellement forwardés sur un trunk.

### 9.4 Autres commandes utiles

| Commande | Utilité |
|---|---|
| `show spanning-tree summary` | mode STP, nombre de ports par état et par VLAN, fonctions activées |
| `show spanning-tree interface gi1/0/2` | rôle et état d'un port dans chaque VLAN |
| `show spanning-tree detail` | compteurs de changements de topologie, dernière TCN et port d'origine |
| `show spanning-tree blockedports` | liste des ports bloqués |

---

## 10. Changements de topologie (TCN)

### 10.1 Le problème

Quand la topologie change, les tables MAC deviennent fausses : une MAC apprise sur un port peut maintenant être joignable par un autre. Avec l'aging par défaut (**300 s**), le trafic irait dans le mauvais sens pendant 5 minutes.

### 10.2 Ce qui déclenche une TCN (802.1D)

- un port qui passe en **forwarding** ;
- un port qui passe de forwarding / learning à **blocking** ou **down**.

> Un port **PortFast** n'en déclenche **pas** (sinon chaque PC allumé ou éteint purgerait les tables).

### 10.3 Le processus

```
         [ROOT] ③ reçoit la TCN → envoie ses BPDU avec le flag TC
           ↑         pendant 35 s, dans TOUT l'arbre
           │ TCN
         [SW2] ② renvoie un TCA à SW3 + relaie la TCN par SON root port
           ↑ TCN   ↓ TCA
         [SW3] ① détecte le changement → TCN par son root port
```

1. Le switch qui détecte le changement envoie une **TCN par son root port** (et la répète à chaque hello tant qu'il n'a pas de TCA).
2. Le switch en amont la reçoit sur un **designated port**, répond par un **TCA** (flag dans une config BPDU) et **relaie** la TCN par son root port.
3. De proche en proche, la TCN arrive au **root**.
4. Le root envoie ses config BPDU avec le **flag TC** pendant **Max Age + Forward Delay = 35 s**.
5. Tous les switches qui reçoivent le flag TC passent leur aging MAC à **15 s**.

| Message | Envoyé par | Vers | Direction |
|---|---|---|---|
| **TCN** | le switch qui détecte, puis chaque relais | le root | **montante**, de root port en root port |
| **TCA** | le switch en amont | l'émetteur de la TCN | descendante, 1 saut |
| **Flag TC** | le root | tous les switches | **descendante**, tout l'arbre |

> Seuls les switches **sur le chemin** vers le root voient passer la TCN. **Tout le monde** reçoit le flag TC.

---

## 11. Pannes directes et indirectes

On reprend le scénario fil rouge, focus sur SW3 :
- root port : **Gi1/0/1** (vers SW1)
- port bloqué : **Gi1/0/2** (vers SW2, qui lui annonce le root avec un coût de 4)

### 11.1 Panne directe : le lien SW1–SW3 tombe

SW3 **voit** son root port passer down.

```
t = 0 s   Gi1/0/1 down. SW3 sait immédiatement que son root port est perdu.
          Gi1/0/2 reçoit déjà des BPDU du root via SW2 (coût 4 + 4 = 8)
          → Gi1/0/2 devient root port.
t = 0     Gi1/0/2 : blocking → listening
t = 15 s  listening → learning
t = 30 s  learning → forwarding
          + TCN envoyée → purge des tables MAC
```

**Convergence : ~30 s** (pas besoin d'attendre le Max Age, la panne est vue directement).

### 11.2 Panne indirecte : SW3 ne reçoit plus de BPDU, mais le lien reste up

Exemple : un problème entre SW1 et SW3 (lien unidirectionnel, équipement intermédiaire…), le port reste up mais les BPDU n'arrivent plus.

```
t = 0 s   Plus de BPDU sur Gi1/0/1, mais le lien est up : SW3 ne se doute de rien.
t = 20 s  Max Age expiré : SW3 invalide l'info du root.
          Ne recevant plus rien de meilleur, il se croit root et s'annonce.
          SW2 reçoit ces BPDU inférieures et répond avec les infos du vrai root (SW1).
          SW3 accepte : Gi1/0/2 devient root port.
t = 20 s  Gi1/0/2 : listening
t = 35 s  learning
t = 50 s  forwarding
```

**Convergence : ~50 s** = Max Age (20) + listening (15) + learning (15).

> Retenir : **directe = 30 s, indirecte = 50 s**. La différence, c'est les 20 s de Max Age pour **se rendre compte** de la panne.

---

## 12. RSTP (802.1w)

### 12.1 Pourquoi RSTP est rapide

| 802.1D | RSTP |
|---|---|
| **attend** (timers) pour être sûr qu'il n'y a pas de boucle | **demande** au voisin (proposal / agreement) |
| recalcule après une panne | a un **remplaçant pré-calculé** (alternate) |
| détecte une panne indirecte en 20 s (Max Age) | **3 hellos manqués = 6 s** |
| seul le root génère les BPDU | **chaque switch** génère ses BPDU à chaque hello |
| TCN qui remonte au root | le switch qui détecte **diffuse lui-même** les TC |
| PortFast = option Cisco | **edge port** intégré au protocole |

Convergence : de **30–50 s** à **environ 1 s** (souvent moins).

### 12.2 États RSTP

| 802.1D | RSTP |
|---|---|
| Disabled | **Discarding** |
| Blocking | **Discarding** |
| Listening | **Discarding** |
| Learning | **Learning** |
| Forwarding | **Forwarding** |

**3 états** : discarding, learning, forwarding.

> Cisco affiche quand même **BLK** dans `show spanning-tree` pour un port discarding.

### 12.3 Rôles RSTP

| Rôle | Description | État |
|---|---|---|
| **Root** | meilleur chemin vers le root | forwarding |
| **Designated** | forwarde vers son segment | forwarding |
| **Alternate** | **chemin de secours vers le root**, via **un autre switch** | discarding |
| **Backup** | **secours d'un designated port** du même switch sur un segment partagé | discarding |
| Disabled | port shutdown | — |

**Alternate vs backup :**

```
ALTERNATE                              BACKUP

  SW1(root)    SW2                          SW1
     │          │                       Gi0/1  Gi0/2
     RP        ALT                        │       │
       \      /                           └─[HUB]─┘
         SW3                                  │
                                           PC  PC
 SW3 reçoit les BPDU de 2 switches     SW1 reçoit SA PROPRE BPDU
 différents → l'autre = alternate      sur Gi0/2 → backup
```

- **Alternate** : très courant (2 uplinks vers 2 switches). Il remplace le **root port**.
- **Backup** : rare, il faut un **média partagé** (hub). Il remplace un **designated port**.

### 12.4 Types de ports RSTP

Ces types décrivent **ce qui est branché en face**, pas le rôle.

| Type | En face | Conséquence |
|---|---|---|
| **Edge** | un hôte (PC, serveur, imprimante) | forwarding **immédiat**. Équivalent de **PortFast**. |
| **Non-edge** | un équipement qui envoie des BPDU | traitement STP normal |
| **Point-to-point** | un switch RSTP en **full-duplex** | proposal / agreement possible → convergence rapide |
| **Shared** | média partagé (**half-duplex**) | retour aux timers lents |

- Un port edge qui **reçoit une BPDU** devient **non-edge**.
- Un uplink vers un switch est à la fois **non-edge** et **point-to-point**.
- Full-duplex = forcément **2 équipements seulement** sur le lien → un seul voisin à qui demander l'accord.

### 12.5 Construction rapide : proposal / agreement (sync)

```
   SW1 (root)
     │ ① proposal  (« je veux passer ce port designated en forwarding »)
     ▼
   SW2  ② sync : bloque tous ses designated ports non-edge
     │        (pour être sûr qu'aucune boucle ne peut se former en aval)
     │  ③ agreement → SW1 passe son port en forwarding immédiatement
     │
     │ ④ SW2 envoie à son tour une proposal sur ses designated ports
     ▼
   SW3  ⑤ sync + agreement… et ainsi de suite jusqu'au bord du réseau
```

- Ça ne marche que sur les liens **point-to-point**.
- Les ports **edge** ne sont pas bloqués par le sync (ils ne peuvent pas créer de boucle).
- La convergence se propage comme une **vague** depuis le root, en quelques millisecondes par saut.

### 12.6 Changement de topologie en RSTP

- Seul un port **non-edge** qui passe en **forwarding** déclenche un TC (perdre un port ne suffit plus).
- Le switch qui détecte :
  1. purge les MAC apprises sur ses ports **non-edge** ;
  2. envoie lui-même des BPDU avec le **flag TC** sur ses ports non-edge (root et designated), pendant **2 × hello** ;
- Chaque voisin fait de même. Plus de passage obligé par le root.

### 12.7 Compatibilité

Si un port RSTP reçoit une **BPDU 802.1D**, il repasse en **802.1D sur ce port** (et perd les avantages de RSTP sur ce lien).

---

## 13. Les variantes : STP, PVST+, RSTP, Rapid PVST+, MST

Deux axes : **l'algorithme** (classique ou rapide) et **le nombre d'instances**.

| Protocole | Norme | Algorithme | Instances | Charge CPU | Partage de charge |
|---|---|---|---|---|---|
| **STP (CST)** | IEEE 802.1D | classique | 1 pour tous les VLAN | faible | non |
| **PVST+** | Cisco | classique | **1 par VLAN** | élevée | oui |
| **RSTP** | IEEE 802.1w | rapide | 1 pour tous | faible | non |
| **Rapid PVST+** | Cisco | rapide | **1 par VLAN** | élevée | oui |
| **MST** | IEEE 802.1s | rapide | **1 par groupe de VLAN** | faible | oui |

### Exemple : 1 switch d'accès, 2 switches de distribution, 100 VLAN

| | Résultat |
|---|---|
| **STP / RSTP** | un seul arbre : l'uplink vers DS2 est bloqué pour **les 100 VLAN**. Un lien inutilisé. |
| **PVST+ / Rapid PVST+** | VLAN 1-50 avec root DS1, VLAN 51-100 avec root DS2 : **les 2 uplinks travaillent**. Mais **100 instances** à calculer. |
| **MST** | instance 1 = VLAN 1-50 (root DS1), instance 2 = VLAN 51-100 (root DS2) : même partage de charge avec **2 instances** seulement. |

> - **PVST+** = une topologie par **VLAN**
> - **MST** = une topologie par **instance** (groupe de VLAN)
> - **Rapid PVST+** = **RSTP appliqué VLAN par VLAN** → `show spanning-tree` affiche `protocol rstp`

```
spanning-tree mode {pvst | rapid-pvst | mst}
```

---

## 14. Configuration essentielle

### 14.1 Fixer le root

```
! Méthode explicite
SW1(config)# spanning-tree vlan 1,10,20 priority 4096
SW2(config)# spanning-tree vlan 1,10,20 priority 8192

! Macros
SW1(config)# spanning-tree vlan 10 root primary
SW2(config)# spanning-tree vlan 10 root secondary
```

| Macro | Effet |
|---|---|
| `root primary` | priorité **24576**, ou **4096 de moins** que le root actuel si ça ne suffit pas |
| `root secondary` | priorité **28672** |

> La macro calcule la valeur **une seule fois**, au moment où tu la tapes. Si un switch plus prioritaire arrive ensuite, elle ne s'adapte pas.

### 14.2 Partage de charge PVST+

```
DS1(config)# spanning-tree vlan 10 root primary
DS1(config)# spanning-tree vlan 20 root secondary
DS2(config)# spanning-tree vlan 20 root primary
DS2(config)# spanning-tree vlan 10 root secondary
```

→ VLAN 10 passe par DS1, VLAN 20 par DS2, et chacun sert de secours à l'autre.

### 14.3 Influencer les ports

```
! Coût (effet LOCAL, sur le choix du root port de CE switch)
SW4(config-if)# spanning-tree vlan 1 cost 2
SW4(config-if)# spanning-tree cost 2            ! tous les VLAN

! Priorité de port (effet chez le VOISIN en aval)
SW1(config-if)# spanning-tree vlan 1 port-priority 64   ! multiple de 16, 0 à 240
```

> - **Coût** : se règle sur le switch **qui choisit** son root port.
> - **Port priority** : se règle sur le switch **émetteur** (en amont), car c'est son port ID que le voisin compare.

### 14.4 Mode et coûts

```
spanning-tree mode rapid-pvst
spanning-tree pathcost method long
```

---

## 15. Exercices corrigés

### Exercice 1 — Élection du root

| Switch | Priorité configurée | MAC |
|---|---|---|
| A | 32768 | 0000.0000.000A |
| B | 28672 | 0000.0000.00FF |
| C | 28672 | 0000.0000.00EE |
| D | 32768 | 0000.0000.0001 |

Qui est root du VLAN 1 ?

<details>
<summary>Réponse</summary>

Priorité la plus basse d'abord : B et C (28672). Égalité → MAC : C (`…00EE`) < B (`…00FF`).
**Root = C.** D a la plus petite MAC mais perd dès le critère priorité.

</details>

---

### Exercice 2 — Root port sur liens parallèles

SW1 est root. SW2 lui est relié par deux câbles, tout par défaut :

```
SW1 Gi0/5  ────  SW2 Gi0/1
SW1 Gi0/3  ────  SW2 Gi0/2
```

Quel est le root port de SW2 ? Comment faire passer le root port sur l'autre lien avec la priorité de port ?

<details>
<summary>Réponse</summary>

Coût identique, émetteur identique (SW1) → priorité de port identique (128) → **numéro de port de l'émetteur** : SW1 Gi0/3 < Gi0/5.
Le root port est le port de SW2 qui reçoit de SW1 Gi0/3 : **SW2 Gi0/2**.

Pour basculer sur l'autre lien : baisser la priorité de **SW1 Gi0/5** (côté émetteur), par exemple `spanning-tree port-priority 64`. La modifier sur SW2 n'a aucun effet sur son choix.
(Autre option : baisser le **coût** de SW2 Gi0/1, réglage local.)

</details>

---

### Exercice 3 — Designated port avec priorité trompeuse

Segment entre SW4 et SW5 :

| | Root path cost | Priorité | MAC |
|---|---|---|---|
| SW4 | 12 | 4096 | 0000.0000.0001 |
| SW5 | 8 | 61440 | ffff.ffff.ffff |

Quel port est designated ?

<details>
<summary>Réponse</summary>

**Celui de SW5.** Le root path cost passe avant le bridge ID : 8 < 12. La priorité 4096 de SW4 ne sert à rien ici.

</details>

---

### Exercice 4 — Calcul de coût

```
ROOT ──10G── SW2 ──1G── SW3 ──100M── SW4
  └──────────────1G──────────────────┘
```

Quel est le root port de SW4 et son root path cost ? (mode short)

<details>
<summary>Réponse</summary>

- Via SW3 : 2 + 4 + 19 = **25**
- Direct vers le root : **4**

Root port = le port **direct** vers le root, coût **4**.

</details>

---

### Exercice 5 — Lecture de sortie

```
SW7# show spanning-tree vlan 30

VLAN0030
  Spanning tree enabled protocol rstp
  Root ID    Priority    24606
             Address     00aa.bbcc.0001
             This bridge is the root
  Bridge ID  Priority    24606  (priority 24576 sys-id-ext 30)
             Address     00aa.bbcc.0001

Interface   Role Sts Cost   Prio.Nbr Type
----------- ---- --- ------ -------- ----
Gi1/0/1     Desg FWD 4      128.1    P2p
Gi1/0/2     Desg FWD 4      128.2    P2p
Gi1/0/10    Desg FWD 4      128.10   P2p Edge
```

1. Quel mode STP tourne ?
2. SW7 est-il root ? Comment est-ce visible de deux façons ?
3. Comment la priorité a-t-elle probablement été configurée ?
4. Que dire de Gi1/0/10 ?

<details>
<summary>Réponse</summary>

1. `protocol rstp` → **Rapid PVST+**.
2. Oui : « This bridge is the root », et Root ID = Bridge ID (même adresse). Tous ses ports sont **Desg**.
3. 24576 = valeur posée par `spanning-tree vlan 30 root primary` (ou `priority 24576`). 24576 + 30 = 24606.
4. **Edge** : port PortFast vers un hôte, forwarding immédiat.

</details>

---

### Exercice 6 — Temps de convergence

Pour chaque cas, donne le temps de convergence approximatif.

1. 802.1D, le root port tombe physiquement, un port bloqué reçoit déjà le root.
2. 802.1D, plus de BPDU sur le root port, lien toujours up.
3. RSTP, le root port tombe, un alternate existe.
4. RSTP, un voisin cesse d'envoyer des BPDU.

<details>
<summary>Réponse</summary>

1. **30 s** (listening 15 + learning 15).
2. **50 s** (Max Age 20 + 15 + 15).
3. **Quasi immédiat** (l'alternate devient root port).
4. Voisin déclaré perdu après **3 hellos = 6 s**, puis bascule rapide.

</details>

---

### Exercice 7 — Rôles RSTP

```
          SW1 (root)
         /         \
     Gi0/1         Gi0/1
      SW2 Gi0/2 ── Gi0/2 SW3
                        Gi0/3 ──[HUB]── Gi0/4 SW3
```

Tous les liens en 1G. SW2 a une MAC plus basse que SW3. Donne le rôle de SW3 Gi0/2, Gi0/3 et Gi0/4 (on suppose Gi0/3 élu designated pour le segment du hub).

<details>
<summary>Réponse</summary>

- **SW3 Gi0/2** : segment SW2–SW3, coûts égaux (4), BID de SW2 plus bas → le port de SW2 est designated, celui de SW3 reçoit le root via un **autre switch** → **alternate**.
- **SW3 Gi0/3** : **designated** pour le segment du hub.
- **SW3 Gi0/4** : reçoit la BPDU de **son propre** Gi0/3 via le hub → **backup**.

</details>

---

### Exercice 8 — TCN

Dans le scénario fil rouge (section 6), le port SW4 Gi1/0/6 (vers SW5) passe down. Par où part la TCN, qui envoie le TCA, et qui reçoit le flag TC ?

<details>
<summary>Réponse</summary>

1. SW4 détecte et envoie une **TCN par son root port** Gi1/0/2 → SW2.
2. SW2 la reçoit sur son designated port Gi1/0/4, renvoie un **TCA** à SW4 et relaie la TCN par son root port Gi1/0/1 → SW1.
3. SW1 (root) envoie ses BPDU avec le **flag TC** pendant 35 s : **tous les switches** (SW2, SW3, SW4, SW5) le reçoivent et passent leur aging MAC à 15 s.

SW3 et SW5 ne voient jamais la TCN, mais reçoivent bien le flag TC.

</details>

---

### Exercice 9 — Conception

Tu as DS1 et DS2 en distribution, des switches d'accès reliés aux deux, et les VLAN 10, 20, 30, 40. Tu veux que les VLAN 10 et 20 passent par DS1, les VLAN 30 et 40 par DS2, avec bascule automatique. Écris la config en Rapid PVST+.

<details>
<summary>Réponse</summary>

```
DS1(config)# spanning-tree mode rapid-pvst
DS1(config)# spanning-tree vlan 10,20 root primary
DS1(config)# spanning-tree vlan 30,40 root secondary

DS2(config)# spanning-tree mode rapid-pvst
DS2(config)# spanning-tree vlan 30,40 root primary
DS2(config)# spanning-tree vlan 10,20 root secondary
```

Sur les switches d'accès : `spanning-tree mode rapid-pvst`, et PortFast + BPDU Guard sur les ports utilisateurs.

</details>

---

## 16. Pièges d'examen

1. **Priorité affichée = configurée + VLAN** (32768 + 10 = 32778).
2. Priorité en **multiples de 4096** uniquement, de 0 à 61440.
3. Le root est élu sur la **priorité d'abord**, la MAC ensuite.
4. Root port : critères 2 à 5 = **valeurs de l'émetteur**, pas les valeurs locales.
5. Liens parallèles : le root port est celui qui reçoit du **plus petit port de l'émetteur**.
6. **Port priority** se règle côté **émetteur** ; **coût** se règle côté **récepteur**.
7. Designated port : le **coût passe avant** le bridge ID.
8. Le root n'a **aucun** root port : tous ses ports sont designated.
9. Cost dans la colonne des interfaces = coût **du port** ; Root Cost en haut = coût **total**.
10. `protocol ieee` = PVST+, `rstp` = Rapid PVST+, `mstp` = MST.
11. 802.1D : **5 états**. RSTP : **3 états**.
12. Blocking **reçoit** les BPDU, **n'en envoie pas**.
13. Listening = BPDU sans apprentissage MAC ; learning = apprentissage MAC sans forwarding.
14. Directe **30 s**, indirecte **50 s**, flag TC **35 s**, aging pendant un TC **15 s**.
15. En 802.1D, seul le **root** génère les config BPDU. En RSTP, **chaque switch**.
16. TCN : **montante** par les root ports. Flag TC : **descendant** depuis le root, à tous.
17. **PortFast** ne déclenche pas de TCN.
18. **Alternate** = secours du root port via un autre switch. **Backup** = secours d'un designated port, segment partagé.
19. **Edge** = PortFast. Un edge qui reçoit une BPDU devient **non-edge** (err-disabled seulement avec BPDU Guard).
20. **Point-to-point** = full-duplex.
21. RSTP : voisin perdu après **3 hellos (6 s)**.
22. RSTP sur un port qui reçoit du **802.1D** → repasse en **802.1D** sur ce port.
23. `root primary` = 24576 (ou 4096 sous le root actuel) ; `root secondary` = 28672.
24. Rapid PVST+ = **RSTP par VLAN**. MST = **instances** regroupant des VLAN.

---

## 17. Aide-mémoire

### Chiffres

| Élément | Valeur |
|---|---|
| Priorité par défaut | 32768 |
| Pas de priorité | 4096 |
| Priorité de port par défaut | 128 (pas de 16) |
| Coûts short | 100 / 19 / 4 / 2 / 1 |
| Coût long | 20 000 000 / Mbps |
| Hello | 2 s |
| Max Age | 20 s |
| Forward Delay | 15 s |
| Panne directe 802.1D | 30 s |
| Panne indirecte 802.1D | 50 s |
| Flag TC 802.1D | 35 s |
| Aging MAC normal / pendant TC | 300 s / 15 s |
| RSTP voisin perdu | 3 hellos = 6 s |
| `root primary` / `secondary` | 24576 / 28672 |

### Commandes

| Tâche | Commande |
|---|---|
| Mode | `spanning-tree mode {pvst \| rapid-pvst \| mst}` |
| Priorité | `spanning-tree vlan <id> priority <0-61440>` |
| Macro root | `spanning-tree vlan <id> root {primary \| secondary}` |
| Coût de port | `spanning-tree [vlan <id>] cost <n>` |
| Priorité de port | `spanning-tree [vlan <id>] port-priority <0-240>` |
| Méthode de coût | `spanning-tree pathcost method {short \| long}` |
| PortFast | `spanning-tree portfast` |
| BPDU Guard | `spanning-tree bpduguard enable` |
| Root par VLAN | `show spanning-tree root` |
| Détail d'un VLAN | `show spanning-tree vlan <id>` |
| Un port, tous VLAN | `show spanning-tree interface <if>` |
| Résumé | `show spanning-tree summary` |
| Historique TC | `show spanning-tree detail` |
| VLAN forwardés sur un trunk | `show interfaces trunk` |

### Algorithme en une page

```
1. ROOT BRIDGE      : plus petit BID (priorité, puis MAC)
2. ROOT PORT        : sur chaque non-root
                      coût → prio émetteur → MAC émetteur → prio port émetteur → n° port émetteur
3. DESIGNATED PORT  : sur chaque segment
                      coût du switch → BID → port ID
4. Le reste         : BLOCKING (802.1D) / ALTERNATE ou BACKUP (RSTP)
```

---

## 18. Annexe : aperçu des chapitres 3 et 4

### Chapitre 3 — Advanced STP Tuning (aperçu)

| Fonction | Rôle | Commande |
|---|---|---|
| **PortFast** | forwarding immédiat sur un port hôte, pas de TCN | `spanning-tree portfast` |
| **BPDU Guard** | err-disable le port s'il reçoit une BPDU | `spanning-tree bpduguard enable` |
| **BPDU Filter** | n'envoie pas / ignore les BPDU (dangereux) | `spanning-tree bpdufilter enable` |
| **Root Guard** | empêche un port de devenir root port (protège le choix du root) | `spanning-tree guard root` |
| **Loop Guard** | empêche un port bloqué de passer forwarding s'il cesse de recevoir des BPDU | `spanning-tree guard loop` |
| **UDLD** | détecte les liens unidirectionnels | `udld enable` |

Port d'accès type :
```
interface gi1/0/10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Chapitre 4 — MST (aperçu)

- Regroupe des VLAN dans des **instances** (MSTI).
- **Instance 0 = IST** : reçoit tous les VLAN non mappés, et porte les échanges entre régions.
- Une **région MST** = des switches avec **même nom, même révision, même mapping VLAN → instance**. La moindre différence crée une autre région.

```
spanning-tree mode mst
spanning-tree mst configuration
 name CAMPUS
 revision 1
 instance 1 vlan 1-50
 instance 2 vlan 51-100
spanning-tree mst 1 root primary
```