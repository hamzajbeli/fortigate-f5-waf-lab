# 🛡️ Sécurisation d'un réseau et d'une application Web

> FortiGate, VPN IPsec, OSPF et WAF F5 BIG-IP sur une maquette EVE-NG,
> avec l'application vulnérable DVWA et des attaques lancées depuis Kali Linux.

## Idée centrale
Un VPN chiffre le lien entre deux sites, mais ne protège pas l'application contre
un hôte autorisé ou compromis. Ce projet le démontre, puis place un WAF devant
l'application pour bloquer les attaques et garder des preuves dans les journaux.

## Contexte
Projet PFA, Université Tek-Up (ING5), année 2026/2027, réalisé à deux.
Rapport complet : [rapport.pdf](rapport.pdf)

## Architecture

![Topologie](topologie.png)

- Un routeur central (FortiGate) relie un Kali externe et deux pare-feu de bordure :
  FGT-2 (Site A) et FGT-3 (Site B).
- Le serveur DVWA est sur le Site A, derrière le F5 BIG-IP, relié à FGT-2 par une DMZ.
- Un second Kali, sur le Site B, simule un poste interne compromis qui passe par le tunnel VPN.
- Le lien direct FGT-2 ↔ DVWA est supprimé : tout le trafic traverse le WAF.

## Ce qui a été réalisé
| Brique | Réalisation |
|---|---|
| Routage | OSPF en Area 0 entre le routeur et les deux FortiGate. Les LAN ne sont pas annoncés, donc le trafic inter-sites passe par le tunnel |
| VPN | Tunnel IPsec route-based, IKEv2, clé pré-partagée, deux sélecteurs de Phase 2 (LAN du Site B et segment Kali) |
| Application | DVWA (Apache, MariaDB, PHP 8.1) sur Ubuntu 22.04, niveau de sécurité Low |
| WAF | F5 BIG-IP 17.5 (LTM + ASM) en reverse proxy, politique `WAF_DVWA`, passée de Transparent à Blocking |

## Tests
Quatre scénarios, avec les mêmes attaques rejouées à chaque fois :

| Scénario | Source | Cible | WAF |
|---|---|---|---|
| A | Kali externe | DVWA directement | Non |
| B | Kali Site B, via le VPN | DVWA directement | Non |
| C | Kali externe | VIP du F5 | Oui |
| D | Kali Site B, via le VPN | VIP du F5 | Oui |

Attaques testées : injection SQL (bypass et UNION), XSS reflected et stored,
injection de commandes, inclusion de fichiers (LFI), CSRF, upload de fichier,
force brute (Hydra).

![Synthèse avant et après le WAF](synthese-waf.png)

**Sans WAF (A et B)** : toutes les attaques aboutissent, y compris depuis le tunnel VPN.
**Avec WAF (C et D)** : les attaques sont bloquées, avec page « Request Rejected »,
Support ID et événement consigné dans les journaux ASM.

### Cas d'étude : injection SQL
| Étape | Configuration | Résultat |
|---|---|---|
| 1 | Requête normale | Réponse de DVWA |
| 2 | Transparent, signature en Staging | Détectée dans le journal, autorisée |
| 3 | Blocking, signature en Staging | Encore autorisée |
| 4 | Blocking, signature Enforced | Bloquée (« Request Rejected ») |

## Difficultés rencontrées
Tunnel inactif faute de Security Policy, Phase 2 refusée (`NO_PROPOSAL_CHOSEN`),
mauvais câblage de l'interface du F5, mémoire insuffisante pour ASM, réponses de DVWA
perdues à cause des routes retour, signature restée en Staging.
Le détail des causes et des solutions est dans le rapport (chapitre 6).

## Limites
- Chiffrement **DES**, imposé par la licence d'évaluation des FortiGate : obsolète,
  à ne jamais utiliser en production
- HTTP uniquement, sans HTTPS ni SSL offloading
- Un seul F5, sans haute disponibilité
- Policies larges (service ALL) sur les FortiGate, acceptables en laboratoire seulement
- SNAT désactivé, ce qui impose des routes retour sur DVWA

## Perspectives
AES-256 avec une licence complète, HTTPS sur le F5, second BIG-IP en haute
disponibilité, affinage de la politique WAF contre les faux positifs, envoi des
journaux vers un SIEM.

## Technologies
FortiGate (FortiOS), IPsec IKEv2, OSPF, F5 BIG-IP (LTM, ASM), DVWA, Kali Linux
(Nmap, Burp Suite, SQLMap, Hydra, Nikto), EVE-NG, VMware.

## Auteurs
- Hamza Jbeli
- Aziz Jebali
