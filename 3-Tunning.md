# ENCOR 350-401 — Chapitre 3 : Advanced STP Tuning

> Cours de révision complet : réglage de la topologie STP (root, coût, priorité de port) et mécanismes de protection (Root Guard, PortFast, BPDU Guard, BPDU Filter, Loop Guard, UDLD).
> Prérequis : le chapitre 2 (élection du root, root ports, designated ports). On réutilise la même topologie fil rouge.

---

## Sommaire

1. [Vue d'ensemble du chapitre](#1-vue-densemble-du-chapitre)
2. [Rappel : la topologie fil rouge](#2-rappel--la-topologie-fil-rouge)
3. [Placer le root bridge](#3-placer-le-root-bridge)
4. [Déplacer root ports et ports bloqués avec le coût](#4-déplacer-root-ports-et-ports-bloqués-avec-le-coût)
5. [Jouer sur la priorité de port](#5-jouer-sur-la-priorité-de-port)
6. [Root Guard](#6-root-guard)
7. [PortFast](#7-portfast)
8. [BPDU Guard](#8-bpdu-guard)
9. [BPDU Filter](#9-bpdu-filter)
10. [Le problème des liens unidirectionnels](#10-le-problème-des-liens-unidirectionnels)
11. [Loop Guard](#11-loop-guard)
12. [UDLD](#12-udld)
13. [Où placer chaque protection](#13-où-placer-chaque-protection)
14. [Vérification et dépannage](#14-vérification-et-dépannage)
15. [Exercices corrigés](#15-exercices-corrigés)
16. [Pièges d'examen](#16-pièges-dexamen)
17. [Aide-mémoire](#17-aide-mémoire)

---

## 1. Vue d'ensemble du chapitre

Le chapitre 2 expliquait **comment STP construit l'arbre tout seul**. Le chapitre 3 répond à deux questions :

| Partie | Question | Outils |
|---|---|---|
| **Topology tuning** | Comment **choisir** la forme de l'arbre ? | priorité (root), coût, priorité de port |
| **Protection mechanisms** | Comment **empêcher** que l'arbre soit cassé (erreur humaine, équipement non prévu, lien défectueux) ? | Root Guard, PortFast, BPDU Guard, BPDU Filter, Loop Guard, UDLD |

> Sans réglage, STP choisit le root au hasard (MAC la plus basse) et le trafic peut emprunter des chemins absurdes. Sans protection, un simple switch branché sous un bureau peut prendre le rôle de root ou créer une boucle.

---

## 2. Rappel : la topologie fil rouge

Tous les liens en 1 Gbps (coût 4), priorités par défaut (VLAN 1 → 32769).

| Switch | MAC |
|---|---|
| SW1 | 0062.ec9d.c500 |
| SW2 | 0081.c4ff.8b00 |
| SW3 | 189c.5d11.9980 |
| SW4 | 70df.2f22.b880 |
| SW5 | bc67.1c5c.9300 |

État de départ (calculé au chapitre 2) :

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

| Switch | Root path cost | Root port |
|---|---|---|
| SW1 | 0 | — |
| SW2 | 4 | Gi1/0/1 |
| SW3 | 4 | Gi1/0/1 |
| SW4 | 8 | Gi1/0/2 |
| SW5 | 8 | Gi1/0/3 |

Ports bloqués : **SW3 Gi1/0/2**, **SW5 Gi1/0/4**, **SW5 Gi1/0/5**.

---

## 3. Placer le root bridge

### 3.1 Pourquoi le placer à la main

- Par défaut, c'est la **MAC la plus basse** qui gagne : souvent le plus vieux switch, parfois un switch d'accès.
- Le root est le **centre de l'arbre** : tout le trafic inter-switch converge vers lui. Il doit être un switch **central et puissant** (cœur ou distribution).
- On désigne toujours un **root secondaire**, pour savoir à l'avance qui prend le relais.

### 3.2 Les deux méthodes

**Priorité explicite** (recommandée, prévisible) :
```
SW1(config)# spanning-tree vlan 1 priority 4096
SW2(config)# spanning-tree vlan 1 priority 8192
```

**Macros** :
```
SW1(config)# spanning-tree vlan 1 root primary
SW2(config)# spanning-tree vlan 1 root secondary
```

| Macro | Valeur posée |
|---|---|
| `root primary` | **24576** si ça suffit à devenir root, sinon **4096 de moins** que la priorité du root actuel |
| `root secondary` | **28672** |

> ⚠️ La macro calcule la valeur **une seule fois**, au moment où tu la tapes. Elle est écrite en dur dans la config (`spanning-tree vlan 1 priority 24576`). Si un switch avec une priorité plus basse arrive plus tard, il deviendra root.

> ⚠️ Si le root actuel est déjà en priorité 0, `root primary` échoue (impossible d'aller plus bas).

### 3.3 Scénario A : on place le root sur SW2

```
SW2(config)# spanning-tree vlan 1 root primary
```

SW1 a la priorité 32769 → 24576 suffit → **SW2 passe en priorité 24577 et devient root**.

**Recalcul complet :**

**Root ports**

| Switch | Chemins | Coût | Root port |
|---|---|---|---|
| SW1 | direct vers SW2 | **4** | **Gi1/0/2** |
| | via SW3 | 8 | |
| SW3 | direct vers SW2 | **4** | **Gi1/0/2** |
| | via SW1 | 8 | |
| SW4 | direct vers SW2 | **4** | **Gi1/0/2** |
| SW5 | via SW3 (annonce 4) | 4 + 4 = **8** | |
| | via SW4 (annonce 4) | 4 + 4 = **8** | |

SW5 est à **égalité de coût** → critère suivant : **bridge ID de l'émetteur**. Priorités égales (32769) → MAC : SW3 `189c…` < SW4 `70df…`.
→ **Root port de SW5 = Gi1/0/3** (vers SW3).

**Designated ports**

| Segment | Décision | Designated | Bloqué |
|---|---|---|---|
| SW1 – SW2 | SW2 est root | SW2 Gi1/0/1 | — |
| SW2 – SW3 | SW2 est root | SW2 Gi1/0/3 | — |
| SW2 – SW4 | SW2 est root | SW2 Gi1/0/4 | — |
| SW1 – SW3 | coûts 4 / 4 → BID : SW1 `0062…` < SW3 | **SW1 Gi1/0/3** | **SW3 Gi1/0/1** |
| SW3 – SW5 | coût 4 < 8 | SW3 Gi1/0/5 | — (RP de SW5) |
| SW4 – SW5 (×2) | coût 4 < 8 | SW4 Gi1/0/5 et Gi1/0/6 | **SW5 Gi1/0/4 et Gi1/0/5** |

**Résultat :**

```
           SW1 Gi1/0/3 DP ─X─ Gi1/0/1 BLK SW3
         RP Gi1/0/2                 Gi1/0/2 RP
              \                    /
         DP Gi1/0/1          Gi1/0/3 DP
                   SW2 (ROOT)
                 DP Gi1/0/4
                    |
                 RP Gi1/0/2           SW3 DP Gi1/0/5
                   SW4                     |
               DP Gi1/0/5 ─X─ Gi1/0/4 BLK  Gi1/0/3 RP
               DP Gi1/0/6 ─X─ Gi1/0/5 BLK      SW5
```

> Leçon : changer **une** priorité reconstruit **tout** l'arbre. Le port bloqué du segment SW2–SW3 a « migré » vers le segment SW1–SW3.

---

## 4. Déplacer root ports et ports bloqués avec le coût

### 4.1 Le principe

Le coût d'un port est **local** : il n'influence que **le switch sur lequel il est configuré**, en modifiant :
- son **root path cost** (s'il s'agit du root port, ou d'un candidat) ;
- donc le choix de **son** root port ;
- et, par ricochet, ce qu'il **annonce** à ses voisins (donc leurs élections de designated port).

```
SW2(config)# interface gi1/0/1
SW2(config-if)# spanning-tree vlan 1 cost 10     ! un seul VLAN
SW2(config-if)# spanning-tree cost 10            ! tous les VLAN
```

- Plage : 1 à 200 000 000.
- Le coût s'applique au port **d'entrée** des BPDU : il compte quand le switch **reçoit** le chemin vers le root par ce port.

### 4.2 Scénario B : on augmente le coût du root port de SW2

On repart de l'état d'origine (SW1 root). Sur SW2 :
```
SW2(config)# interface gi1/0/1
SW2(config-if)# spanning-tree vlan 1 cost 10
```

**SW2 recalcule son root port :**

| Chemin de SW2 | Coût |
|---|---|
| Gi1/0/1 direct vers SW1 | **10** (nouveau coût) |
| Gi1/0/3 via SW3 (annonce 4) | 4 + 4 = **8** ✅ |

→ **Nouveau root port de SW2 = Gi1/0/3**, root path cost = **8**.

**Effet domino :**

| Élément | Avant | Après | Pourquoi |
|---|---|---|---|
| SW2 Gi1/0/1 | RP | **bloqué** | segment SW1–SW2 : SW1 (coût 0) est designated, et ce n'est plus le root port de SW2 |
| Segment SW2–SW3 | SW2 DP, SW3 bloqué | **SW3 Gi1/0/2 DP**, SW2 Gi1/0/3 = RP | SW3 annonce 4, SW2 annonce 8 : SW3 gagne |
| SW4 root port | Gi1/0/2, coût 8 | Gi1/0/2, coût **12** | via SW2 : 8 + 4 = 12 ; via SW5 : 8 + 4 = 12 → égalité → BID : SW2 `0081…` < SW5 → reste Gi1/0/2 |
| Segment SW4–SW5 | SW4 DP, SW5 bloqué | **SW5 DP**, **SW4 Gi1/0/5 et Gi1/0/6 bloqués** | SW5 annonce 8, SW4 annonce 12 : SW5 gagne |

**Résultat :**

```
                     SW1 (ROOT)
               DP Gi1/0/2   Gi1/0/3 DP
                X                  \
        BLK Gi1/0/1             Gi1/0/1 RP
           SW2 RP Gi1/0/3 ──── Gi1/0/2 DP SW3
         DP Gi1/0/4                 Gi1/0/5 DP
           |                           |
         RP Gi1/0/2                 Gi1/0/3 RP
           SW4 BLK Gi1/0/5 ─X─ Gi1/0/4 DP SW5
               BLK Gi1/0/6 ─X─ Gi1/0/5 DP
```

Ports bloqués : **SW2 Gi1/0/1**, **SW4 Gi1/0/5**, **SW4 Gi1/0/6**.

> Leçon : un changement de coût sur **un seul port** a déplacé **trois** ports bloqués. Toujours recalculer de proche en proche : root ports d'abord, puis designated ports.

### 4.3 Vérifier

```
SW2# show spanning-tree vlan 1

VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0062.ec9d.c500
             Cost        8
             Port        3 (GigabitEthernet1/0/3)
...
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ------
Gi1/0/1             Altn BLK 10        128.1    P2p
Gi1/0/3             Root FWD 4         128.3    P2p
Gi1/0/4             Desg FWD 4         128.4    P2p
```

- `Cost 8` en haut = root path cost (total).
- `10` dans la colonne Cost de Gi1/0/1 = le coût **configuré sur ce port**.

---

## 5. Jouer sur la priorité de port

### 5.1 Le principe

La priorité de port fait partie du **port ID** (priorité . numéro). Elle sert de départage **après** le coût et le bridge ID :
- élection du **root port** → c'est le port ID **de l'émetteur** qui compte ;
- élection du **designated port** → c'est le port ID du switch candidat.

```
SW1(config)# interface gi1/0/5
SW1(config-if)# spanning-tree vlan 1 port-priority 64
SW1(config-if)# spanning-tree port-priority 64        ! tous les VLAN
```

- Valeur par défaut : **128**.
- Plage : **0 à 240, par pas de 16**.
- La plus basse est préférée.

### 5.2 Scénario C : liens parallèles

SW1 est root. SW2 lui est relié par deux câbles :

```
SW1 Gi1/0/5  ────  SW2 Gi1/0/1
SW1 Gi1/0/3  ────  SW2 Gi1/0/2
```

**Par défaut :** coût égal, émetteur égal (SW1), priorité égale (128) → numéro de port de l'émetteur : SW1 Gi1/0/3 < Gi1/0/5.
→ **Root port de SW2 = Gi1/0/2** (celui qui reçoit de SW1 Gi1/0/3).

**Objectif : faire passer SW2 par l'autre lien (Gi1/0/1).**

| Option | Où | Commande | Effet |
|---|---|---|---|
| ❌ priorité de port sur SW2 | SW2 Gi1/0/1 | `port-priority 64` | **aucun** effet sur le root port de SW2 (ce n'est pas l'émetteur) |
| ✅ priorité de port sur SW1 | SW1 Gi1/0/5 | `port-priority 64` | SW2 voit 64.5 < 128.3 → root port = **Gi1/0/1** |
| ✅ coût sur SW2 | SW2 Gi1/0/1 | `spanning-tree cost 2` | coût 2 < 4 → root port = **Gi1/0/1** |

> ⚠️ **Le piège n°1 du chapitre** :
> - **Coût** → se règle sur le switch **qui choisit** (en aval).
> - **Priorité de port** → se règle sur le switch **qui émet** (en amont).

---

## 6. Root Guard

### 6.1 Le problème

Un switch non prévu (un switch d'un client, d'un labo, ou mal configuré) est branché et annonce une **priorité plus basse** que ton root. Il **devient root** : tout l'arbre se reconstruit autour de lui, le trafic passe par un switch d'accès, voire sort du réseau.

### 6.2 Le fonctionnement

Root Guard s'active sur un **designated port** et dit : **« ce port ne doit jamais devenir root port »**.

- Si le port reçoit une **BPDU supérieure** (meilleure que le root actuel), il passe en état **root-inconsistent** : il est **bloqué** (aucun forwarding).
- Dès que les BPDU supérieures **cessent**, le port **se rétablit automatiquement** (pas d'err-disable).

```
SW1(config)# interface gi1/0/2
SW1(config-if)# spanning-tree guard root
```

### 6.3 Où le placer

Sur les **designated ports** des switches de **cœur / distribution** qui pointent **vers l'aval** (accès, clients, partenaires) : là où un root ne devrait **jamais** apparaître.

```
          [ROOT / Core]
         ┌──Root Guard──┐
     [Distribution]  [Distribution]
         ├──Root Guard──┤
      [Access]        [Access]
```

> Ne **jamais** le mettre sur un root port ou sur un lien vers le root légitime : ce port doit pouvoir recevoir les BPDU du root.

### 6.4 Vérifier

```
SW1# show spanning-tree inconsistentports

Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
VLAN0001             GigabitEthernet1/0/2   Root Inconsistent

Number of inconsistent ports (segments) in the system : 1
```

Dans `show spanning-tree`, le port apparaît en **BKN\*** avec le type **ROOT_Inc**.

Log typique :
```
%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port GigabitEthernet1/0/2 on VLAN0001.
```

---

## 7. PortFast

### 7.1 Le problème

Un PC branché sur un port en 802.1D attend **30 s** (listening + learning) avant d'avoir du réseau. Conséquences : échec DHCP, PXE qui ne boote pas, ouverture de session lente. Et chaque allumage / extinction de PC déclenche une **TCN**, qui purge les tables MAC de tout le réseau.

### 7.2 Le fonctionnement

| Effet | Détail |
|---|---|
| Forwarding immédiat | le port saute listening et learning |
| Pas de TCN | les changements d'état du port ne génèrent pas de TCN |
| Envoie toujours des BPDU | PortFast **ne bloque pas** les BPDU |
| Garde-fou intégré | s'il **reçoit** une BPDU, il perd son statut PortFast (opérationnel) et redevient un port STP normal |

> En RSTP, un port PortFast correspond à un **edge port**.

### 7.3 Configuration

```
! Par interface (port access)
SW1(config-if)# spanning-tree portfast

! Par interface, port trunk vers un hôte (hyperviseur, serveur multi-VLAN)
SW1(config-if)# spanning-tree portfast trunk

! Global : active PortFast sur TOUS les ports access
SW1(config)# spanning-tree portfast default

! Désactiver sur un port précis quand le global est actif
SW1(config-if)# spanning-tree portfast disable
```

Message affiché à l'activation :
```
%Warning: portfast should only be enabled on ports connected to a single
 host. Connecting hubs, concentrators, switches, bridges, etc... to this
 interface  when portfast is enabled, can cause temporary bridging loops.
```

> ⚠️ `spanning-tree portfast default` ne s'applique qu'aux ports en mode **access**, pas aux trunks.

---

## 8. BPDU Guard

### 8.1 Le problème

PortFast seul laisse passer un switch branché sur une prise utilisateur : le port perd simplement son statut PortFast, et le switch non maîtrisé **entre dans la topologie**.

### 8.2 Le fonctionnement

Dès qu'un port protégé par BPDU Guard **reçoit une BPDU**, il passe en **err-disabled** (coupé).

```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi1/0/10 with BPDU Guard enabled.
 Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi1/0/10, putting Gi1/0/10 in err-disable state
```

### 8.3 Configuration

```
! Par interface (indépendant de PortFast)
SW1(config-if)# spanning-tree bpduguard enable

! Global : sur tous les ports où PortFast est actif
SW1(config)# spanning-tree portfast bpduguard default
```

| Méthode | S'applique à |
|---|---|
| Interface | ce port, **PortFast ou non** |
| Global (`portfast bpduguard default`) | uniquement les ports **PortFast** opérationnels |

### 8.4 Remettre le port en service

**Manuellement :**
```
SW1(config)# interface gi1/0/10
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

**Automatiquement (errdisable recovery) :**
```
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300      ! défaut : 300 s
```

> Si le switch fautif est toujours branché, le port retombe en err-disabled dès la première BPDU après la remise en service.

### 8.5 Vérifier

```
SW1# show interfaces status err-disabled

Port      Name     Status       Reason               Err-disabled Vlans
Gi1/0/10           err-disabled bpduguard

SW1# show errdisable recovery
ErrDisable Reason            Timer Status
-----------------            --------------
bpduguard                    Enabled
...
Timer interval: 300 seconds
```

### 8.6 Le combo standard des ports utilisateurs

```
interface range gi1/0/1 - 44
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## 9. BPDU Filter

### 9.1 Le fonctionnement

BPDU Filter **supprime les échanges de BPDU** sur un port. Son comportement dépend **complètement** de la façon dont il est activé.

| | **Global** | **Interface** |
|---|---|---|
| Commande | `spanning-tree portfast bpdufilter default` | `spanning-tree bpdufilter enable` |
| S'applique à | ports **PortFast** uniquement | ce port, quel que soit son mode |
| Envoi de BPDU | quelques BPDU à la montée du lien, puis plus rien | **aucune** BPDU |
| Réception d'une BPDU | le port **perd PortFast et le filtre**, redevient un port STP normal | la BPDU est **ignorée** |
| Risque | faible : il y a un garde-fou | **élevé** : STP est **désactivé** sur ce port |

### 9.2 Pourquoi l'interface est dangereuse

Avec `spanning-tree bpdufilter enable` sur un port :
- il n'envoie rien, il ignore tout ;
- si un switch est branché derrière et qu'une boucle existe, **STP ne la verra jamais** → broadcast storm.

> Usage typique : frontière administrative (lien vers un opérateur ou un client qui ne doit pas participer à ton STP). À manier avec une extrême prudence.

### 9.3 BPDU Guard + BPDU Filter sur le même port

Si les deux sont actifs, le filtre **empêche** BPDU Guard de voir les BPDU : **BPDU Guard ne se déclenche jamais**. C'est une erreur de config classique.

---

## 10. Le problème des liens unidirectionnels

### 10.1 Le scénario

Une fibre a deux brins : un pour émettre, un pour recevoir. Si **un seul brin** casse, le lien peut rester **up** dans un sens tout en ne transmettant plus dans l'autre.

```
SW2 (DP) ════════════> SW3 (port bloqué)
         <════X═══════
```

Supposons que le brin **SW2 → SW3** soit coupé :
1. SW3 avait ici un **port bloqué** (alternate) qui recevait les BPDU de SW2.
2. Il ne reçoit **plus aucune BPDU**, mais le lien reste **up** (il continue d'émettre).
3. Après **Max Age (20 s)**, SW3 considère que l'info est perdue.
4. Ne recevant rien de mieux, il passe ce port **designated → forwarding**.
5. Résultat : **boucle L2**, alors qu'aucun port n'est tombé.

> STP suppose que si on ne reçoit plus de BPDU, c'est que le voisin a disparu. Un lien unidirectionnel trompe cette hypothèse.

Deux protections : **Loop Guard** (côté STP) et **UDLD** (côté lien).

---

## 11. Loop Guard

### 11.1 Le fonctionnement

Loop Guard s'active sur les ports **non-designated** (root ports, alternate / bloqués) et dit : **« si ce port arrête de recevoir des BPDU, il ne passe pas en forwarding »**.

- Au lieu de passer designated / forwarding après Max Age, le port passe en **loop-inconsistent** (bloqué).
- Dès que les BPDU **reviennent**, il se **rétablit automatiquement**.

```
! Par interface
SW3(config-if)# spanning-tree guard loop

! Global : sur tous les ports point-to-point
SW3(config)# spanning-tree loopguard default
```

### 11.2 Vérifier

```
SW3# show spanning-tree inconsistentports

Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
VLAN0001             GigabitEthernet1/0/2   Loop Inconsistent
```

Log :
```
%SPANTREE-2-LOOPGUARD_BLOCK: Loop guard blocking port GigabitEthernet1/0/2 on VLAN0001.
```

### 11.3 Root Guard vs Loop Guard

| | **Root Guard** | **Loop Guard** |
|---|---|---|
| Se place sur | **designated** ports (vers l'aval) | **root** et **alternate** ports (vers le root) |
| Se déclenche quand | le port **reçoit** une BPDU **supérieure** | le port **ne reçoit plus** de BPDU |
| État | root-inconsistent | loop-inconsistent |
| Protège contre | un **root non prévu** | une **boucle** due à un lien unidirectionnel |
| Récupération | automatique | automatique |

> ⚠️ Root Guard et Loop Guard sont **incompatibles sur un même port** : l'un est fait pour les designated ports, l'autre pour les ports non-designated.

---

## 12. UDLD

### 12.1 Le fonctionnement

**UDLD (UniDirectional Link Detection)** est un protocole **Cisco** de couche 2 qui vérifie que **chaque lien fonctionne dans les deux sens** :
- chaque switch envoie des messages UDLD contenant **son identité et son port** ;
- chaque switch renvoie dans ses messages ce qu'il a **reçu** du voisin ;
- si un switch ne se voit pas dans les messages du voisin, le lien est **unidirectionnel**.

Intervalle des messages : **15 s** par défaut.

> UDLD doit être activé **des deux côtés** du lien.

### 12.2 Les deux modes

| Mode | Réaction à un lien unidirectionnel |
|---|---|
| **Normal** | détecte et **génère un log**, le port reste actif |
| **Aggressive** | tente de rétablir le contact (**8 essais**, un par seconde), puis passe le port en **err-disabled** |

> Le mode **aggressive** est celui qui protège réellement contre la boucle.

### 12.3 Configuration

```
! Global : sur tous les ports FIBRE
SW1(config)# udld enable
SW1(config)# udld aggressive

! Par interface (cuivre ou fibre)
SW1(config-if)# udld port
SW1(config-if)# udld port aggressive
```

Remise en service après err-disable :
```
SW1# udld reset
! ou shutdown / no shutdown, ou errdisable recovery cause udld
```

### 12.4 Vérifier

```
SW1# show udld neighbors
Port      Device Name   Device ID   Port ID    Neighbor State
--------  -----------   ---------   -------    --------------
Gi1/0/49  SW2           1           Gi1/0/49   Bidirectional

SW1# show udld gi1/0/49
```

### 12.5 Loop Guard vs UDLD

| | **Loop Guard** | **UDLD** |
|---|---|---|
| Niveau | STP | lien (L2, protocole dédié) |
| Détecte | l'absence de BPDU | un lien unidirectionnel, directement |
| Standard | fonction Cisco sur STP | protocole Cisco |
| Réaction | port loop-inconsistent (par VLAN) | log (normal) ou err-disabled (aggressive), tout le port |
| Délai | après Max Age | dépend des messages UDLD |

> Les deux sont **complémentaires** : on les active souvent ensemble sur les liens fibre inter-switch.

---

## 13. Où placer chaque protection

```
                    [CORE / ROOT]
                         │
     Root Guard ─► DP    │    DP ◄─ Root Guard
                         │
         ┌───────────────┴───────────────┐
   [DISTRIBUTION]                  [DISTRIBUTION]
      RP / ALT ◄─ Loop Guard + UDLD (liens fibre)
         │                               │
   Root Guard ─► DP                 DP ◄─ Root Guard
         │                               │
      [ACCESS]                       [ACCESS]
      RP / ALT ◄─ Loop Guard + UDLD
         │
   Ports utilisateurs ◄─ PortFast + BPDU Guard
         │
        PC
```

| Emplacement | Protection | Pourquoi |
|---|---|---|
| Ports vers les **PC / imprimantes / serveurs** | **PortFast + BPDU Guard** | forwarding immédiat, et coupure si un switch est branché |
| **Designated ports** cœur / distribution vers l'aval | **Root Guard** | personne en aval ne doit devenir root |
| **Root ports / alternate** des uplinks | **Loop Guard** | un port bloqué ne doit pas s'ouvrir s'il perd les BPDU |
| **Liens fibre** inter-switch | **UDLD aggressive** | détecter un brin coupé |
| Frontière administrative (opérateur, client) | **BPDU Filter** (avec prudence) | ne pas échanger de BPDU avec un domaine externe |

---

## 14. Vérification et dépannage

| Commande | Ce qu'elle montre |
|---|---|
| `show spanning-tree vlan <id>` | rôles, états, coûts ; ports **BKN\*** en cas d'incohérence |
| `show spanning-tree interface <if> detail` | PortFast, BPDU Guard, Root/Loop Guard actifs, BPDU envoyées / reçues |
| `show spanning-tree summary` | fonctions activées globalement (PortFast default, BPDU Guard default, Loop Guard default…) |
| `show spanning-tree inconsistentports` | ports **root-inconsistent** et **loop-inconsistent** |
| `show interfaces status err-disabled` | ports coupés et la cause (bpduguard, udld…) |
| `show errdisable recovery` | causes avec récupération automatique et intervalle |
| `show udld neighbors` | état UDLD par port |

Exemple de `show spanning-tree summary` :
```
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0001
EtherChannel misconfig guard            is enabled
Extended system ID                      is enabled
Portfast Default                        is enabled
PortFast BPDU Guard Default             is enabled
Portfast BPDU Filter Default            is disabled
Loopguard Default                       is enabled
UplinkFast                              is disabled
BackboneFast                            is disabled
Pathcost method used                    is short
```

### Diagnostic rapide

| Symptôme | Cause probable | Vérification |
|---|---|---|
| Port utilisateur coupé, voyant orange | BPDU Guard (switch branché) | `show interfaces status err-disabled` |
| Port bloqué sans raison apparente, BKN\* | Root Guard ou Loop Guard | `show spanning-tree inconsistentports` |
| Root qui change tout seul | pas de Root Guard, switch non prévu | `show spanning-tree root`, `show spanning-tree detail` |
| Broadcast storm sans port tombé | lien unidirectionnel ou BPDU Filter en interface | UDLD, Loop Guard, config des filtres |
| PC qui met 30 s à avoir une IP | pas de PortFast | `show spanning-tree interface <if> detail` |

---

## 15. Exercices corrigés

### Exercice 1 — Macro root primary

Le root actuel du VLAN 10 a une priorité configurée de **16384**. Tu tapes `spanning-tree vlan 10 root primary` sur SW2. Quelle priorité est configurée, et que montre `show spanning-tree vlan 10` dans le champ Priority du Bridge ID ?

<details>
<summary>Réponse</summary>

24576 ne suffit pas (16384 est plus bas) → la macro pose **4096 de moins** que le root actuel : 16384 − 4096 = **12288**.
Affiché : 12288 + 10 = **12298**.

</details>

---

### Exercice 2 — Où régler quoi

SW1 est root. SW2 a deux liens vers SW1 : SW2 Gi0/1 ↔ SW1 Gi0/7 et SW2 Gi0/2 ↔ SW1 Gi0/4. Tout est par défaut.

1. Quel est le root port de SW2 ?
2. Tu veux que SW2 utilise Gi0/1. Donne deux méthodes.

<details>
<summary>Réponse</summary>

1. Tout est égal jusqu'au numéro de port de l'émetteur : SW1 Gi0/4 < Gi0/7 → root port = **SW2 Gi0/2**.
2. - **Coût**, sur SW2 : `interface gi0/1` → `spanning-tree cost 2`.
   - **Priorité de port**, sur **SW1** : `interface gi0/7` → `spanning-tree port-priority 64`.

Mettre la priorité de port sur SW2 Gi0/1 ne change rien.

</details>

---

### Exercice 3 — Effet domino

Topologie fil rouge d'origine (SW1 root). Sur SW3, tu configures `spanning-tree vlan 1 cost 19` sur Gi1/0/1. Quels sont les nouveaux root port et root path cost de SW3 ? Quel port est bloqué sur le segment SW2–SW3 ?

<details>
<summary>Réponse</summary>

SW3 :
- Gi1/0/1 direct : **19**
- Gi1/0/2 via SW2 (annonce 4) : 4 + 4 = **8** ✅

→ Root port de SW3 = **Gi1/0/2**, root path cost **8**.

Segment SW2–SW3 : SW2 annonce 4, SW3 annonce 8 → **SW2 Gi1/0/3 designated**, SW3 Gi1/0/2 est son root port (forwarding).
Segment SW1–SW3 : SW1 designated (coût 0), **SW3 Gi1/0/1 bloqué**.

Et SW5 : via SW3, 8 + 4 = 12 ; via SW4, 8 + 4 = 12 → égalité → BID de l'émetteur : SW3 `189c…` < SW4 `70df…` → SW5 garde Gi1/0/3. Segment SW4–SW5 : SW4 (8) contre SW5 (12) → SW4 reste designated.

</details>

---

### Exercice 4 — Quelle protection ?

Pour chaque situation, choisis la bonne fonction.

1. Un stagiaire branche un petit switch sur la prise de son bureau.
2. Un client relié à ton cœur de réseau annonce une priorité de 0.
3. Une fibre entre deux distributions perd un brin.
4. Les PC mettent 30 s à obtenir une adresse DHCP.
5. Tu dois relier ton réseau à celui d'un opérateur sans échanger de BPDU.

<details>
<summary>Réponse</summary>

1. **BPDU Guard** (avec PortFast) : port err-disabled.
2. **Root Guard** sur le port qui fait face au client : root-inconsistent.
3. **UDLD aggressive** (détection directe) et/ou **Loop Guard** (le port bloqué ne s'ouvre pas).
4. **PortFast**.
5. **BPDU Filter** en interface, en connaissant le risque.

</details>

---

### Exercice 5 — Lecture de logs

```
%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port GigabitEthernet1/0/24 on VLAN0020.
```

1. Que s'est-il passé ?
2. Le port est-il err-disabled ?
3. Comment se rétablit-il ?

<details>
<summary>Réponse</summary>

1. Gi1/0/24 a reçu une **BPDU supérieure** dans le VLAN 20 : un switch derrière ce port prétend être un meilleur root.
2. **Non** : il est en **root-inconsistent** (bloqué pour le VLAN 20), pas err-disabled.
3. **Automatiquement**, dès que les BPDU supérieures cessent (correction de la priorité en face, ou débranchement).

</details>

---

### Exercice 6 — Global vs interface

Sur un switch : `spanning-tree portfast default` et `spanning-tree portfast bpduguard default`. Le port Gi1/0/48 est un **trunk** vers un autre switch, sans config STP particulière. Que se passe-t-il s'il reçoit des BPDU ?

<details>
<summary>Réponse</summary>

**Rien de particulier.** `portfast default` ne s'applique qu'aux ports **access** : Gi1/0/48 (trunk) n'est pas PortFast. Et `portfast bpduguard default` ne protège que les ports **PortFast**. Le trunk participe donc normalement à STP.

</details>

---

### Exercice 7 — Piège BPDU Filter

Un port utilisateur a `spanning-tree portfast`, `spanning-tree bpduguard enable` **et** `spanning-tree bpdufilter enable`. Quelqu'un y branche un switch. Que se passe-t-il ?

<details>
<summary>Réponse</summary>

Le **BPDU Filter en interface** ignore toutes les BPDU reçues : **BPDU Guard ne les voit jamais** et ne se déclenche pas. Le switch est accepté sans que STP le voie : risque de **boucle**. Il faut retirer le filtre.

</details>

---

### Exercice 8 — Conception

Écris la config des ports pour :
- SW-ACC1 : ports Gi1/0/1-44 vers des PC (VLAN 10), uplinks fibre Gi1/0/49-50 vers DS1 et DS2 ;
- DS1 : port Gi1/0/10 vers SW-ACC1.

<details>
<summary>Réponse</summary>

```
SW-ACC1(config)# interface range gi1/0/1 - 44
SW-ACC1(config-if-range)# switchport mode access
SW-ACC1(config-if-range)# switchport access vlan 10
SW-ACC1(config-if-range)# spanning-tree portfast
SW-ACC1(config-if-range)# spanning-tree bpduguard enable

SW-ACC1(config)# interface range gi1/0/49 - 50
SW-ACC1(config-if-range)# switchport mode trunk
SW-ACC1(config-if-range)# spanning-tree guard loop
SW-ACC1(config-if-range)# udld port aggressive

DS1(config)# interface gi1/0/10
DS1(config-if)# switchport mode trunk
DS1(config-if)# spanning-tree guard root
DS1(config-if)# udld port aggressive
```

- Ports PC : PortFast + BPDU Guard.
- Uplinks de l'accès (root port / alternate) : Loop Guard + UDLD.
- Côté distribution (designated vers l'accès) : Root Guard + UDLD.

</details>

---

## 16. Pièges d'examen

1. `root primary` = **24576**, ou **4096 de moins** que le root actuel. `root secondary` = **28672**.
2. La macro est **calculée une fois**, puis écrite en dur dans la config.
3. **Coût** = réglage **local** (switch qui choisit). **Port priority** = réglage chez l'**émetteur**.
4. Port priority : défaut **128**, de **0 à 240** par **pas de 16**.
5. Un changement de coût ou de priorité peut déplacer **plusieurs** ports bloqués.
6. **Root Guard** : sur les **designated** ports vers l'aval, déclenché par une **BPDU supérieure**, état **root-inconsistent**, récupération **automatique**.
7. **PortFast** : forwarding immédiat, **pas de TCN**, envoie **toujours** des BPDU, perd son statut s'il en **reçoit** une.
8. `spanning-tree portfast default` : ports **access** seulement. Pour un trunk vers un hôte : `spanning-tree portfast trunk`.
9. **BPDU Guard** : réception d'une BPDU → **err-disabled**. Récupération : shut / no shut ou **errdisable recovery** (300 s par défaut).
10. BPDU Guard **global** = ports PortFast seulement. BPDU Guard **interface** = indépendant de PortFast.
11. **BPDU Filter global** : ports PortFast, perd le filtre s'il reçoit une BPDU (sûr).
12. **BPDU Filter interface** : ignore tout, n'envoie rien → **STP désactivé** sur le port (dangereux).
13. BPDU Filter interface **neutralise** BPDU Guard.
14. Lien unidirectionnel : un port bloqué qui ne reçoit plus de BPDU **s'ouvre après Max Age** → boucle.
15. **Loop Guard** : sur **root / alternate** ports, déclenché par l'**absence** de BPDU, état **loop-inconsistent**, récupération **automatique**.
16. Root Guard et Loop Guard : **incompatibles sur le même port**.
17. **UDLD** : protocole **Cisco**, à activer **des deux côtés**. **Normal** = log ; **aggressive** = err-disabled après **8 essais**.
18. `udld enable` (global) = ports **fibre** seulement.
19. `show spanning-tree inconsistentports` pour Root Guard et Loop Guard ; `show interfaces status err-disabled` pour BPDU Guard et UDLD.

---

## 17. Aide-mémoire

### Chiffres

| Élément | Valeur |
|---|---|
| `root primary` | 24576, ou root actuel − 4096 |
| `root secondary` | 28672 |
| Port priority | 128 par défaut, 0–240, pas de 16 |
| Coût de port | 1 à 200 000 000 |
| Errdisable recovery | 300 s par défaut |
| UDLD message interval | 15 s |
| UDLD aggressive | 8 essais avant err-disable |

### Les états à ne pas confondre

| Fonction | État du port | Récupération |
|---|---|---|
| Root Guard | root-inconsistent (BKN\*) | automatique |
| Loop Guard | loop-inconsistent (BKN\*) | automatique |
| BPDU Guard | err-disabled | manuelle ou errdisable recovery |
| UDLD aggressive | err-disabled | `udld reset`, manuelle ou errdisable recovery |

### Commandes

| Tâche | Commande |
|---|---|
| Priorité | `spanning-tree vlan <id> priority <n>` |
| Macros | `spanning-tree vlan <id> root {primary \| secondary}` |
| Coût | `spanning-tree [vlan <id>] cost <n>` |
| Priorité de port | `spanning-tree [vlan <id>] port-priority <n>` |
| Root Guard | `spanning-tree guard root` |
| PortFast | `spanning-tree portfast` / `portfast trunk` / `portfast default` |
| BPDU Guard | `spanning-tree bpduguard enable` / `spanning-tree portfast bpduguard default` |
| BPDU Filter | `spanning-tree bpdufilter enable` / `spanning-tree portfast bpdufilter default` |
| Loop Guard | `spanning-tree guard loop` / `spanning-tree loopguard default` |
| UDLD | `udld enable` / `udld aggressive` / `udld port [aggressive]` |
| Errdisable recovery | `errdisable recovery cause <cause>` + `errdisable recovery interval <s>` |
| Incohérences | `show spanning-tree inconsistentports` |
| Ports coupés | `show interfaces status err-disabled` |
| Fonctions globales | `show spanning-tree summary` |
| Détail d'un port | `show spanning-tree interface <if> detail` |
| UDLD | `show udld neighbors` |

### En une phrase par fonction

- **Root Guard** : « personne derrière ce port ne deviendra root ».
- **PortFast** : « c'est un PC, ouvre tout de suite ».
- **BPDU Guard** : « si un switch apparaît ici, coupe ».
- **BPDU Filter** : « ne parle pas STP sur ce port » (attention).
- **Loop Guard** : « si les BPDU disparaissent, ne t'ouvre pas ».
- **UDLD** : « vérifie que le lien marche dans les deux sens ».