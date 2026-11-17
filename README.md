# 🏐 Simulation d'un match de volley-ball — NetLogo
 
Simulation multi-agents d'un match de volley-ball entre deux équipes de six joueurs, réalisée en **NetLogo** dans le cadre du module *Introduction à la programmation Agent* (L2 Informatique, Faculté de Sciences, Vandœuvre-lès-Nancy).
 
Chaque joueur et la balle sont des agents autonomes. Le modèle sert à étudier l'influence des caractéristiques des joueurs (puissance, précision, taux d'échec, type de service, perception) sur l'issue d'un match.
 
**Auteurs :** Dorian Berger, Hector Bos
 
---
 
## Sommaire
 
- [Aperçu](#aperçu)
- [Installation et lancement](#installation-et-lancement)
- [Paramètres](#paramètres)
- [Fonctionnement du modèle](#fonctionnement-du-modèle)
- [Résultats](#résultats)
- [Limites du modèle](#limites-du-modèle)
- [Structure du dépôt](#structure-du-dépôt)
---
 
## Aperçu
 
- **Terrain** : vue du dessus, avec le filet (ligne blanche), les lignes des 3 mètres et les limites du terrain.
- **Équipes** : *Domicile* en rouge (libéro en rose) et *Extérieur* en vert (libéro en jaune).
- **Postes** : R1 à R6. R1 est le serveur, R3 le passeur, R2 et R4 les attaquants, R5 et R6 les défenseurs arrière.
- **Règles reprises** : rotation des joueurs après la reprise du service, trois touches au maximum par camp, set gagné à 25 points avec 2 points d'écart.
- **Services** : smash, flottant (trajectoire en zigzag) et normal.
## Installation et lancement
 
### Prérequis
 
- [NetLogo](https://ccl.northwestern.edu/netlogo/) **7.0 ou plus récent** (le modèle est au format `.nlogox`).
### Lancer un match
 
1. Ouvrir `Projet_Volley_BERGER_BOS.nlogox` dans NetLogo.
2. Régler les curseurs (voir [Paramètres](#paramètres)).
3. Cliquer sur **setup** pour placer les joueurs et tirer au sort l'équipe qui sert.
4. Cliquer sur **go** pour lancer le match. Il s'arrête de lui-même dès qu'une équipe a gagné.
Les compteurs *Domicile* et *Extérieur* affichent le score en direct. Les événements (type de service, réceptions ratées, victoire) s'affichent dans le *Command Center*.
 
### Simuler plusieurs matchs
 
Pour obtenir des statistiques, tape dans le *Command Center* (observer) :
 
```netlogo
simuler-n 100
```
 
Cette commande enchaîne 100 matchs avec les paramètres courants, puis affiche :
 
- le score de chaque match et le cumul des victoires ;
- le nombre moyen de ticks par match ;
- le bilan des victoires Rouge / Vert.
Le graphique **Proportion de victoires** est mis à jour à la fin de la série.
 
## Paramètres
 
| Curseur | Portée | Rôle |
|---|---|---|
| `puissance-max-dom` / `-ext` | par équipe | Puissance maximale des joueurs. Elle augmente la vitesse de la balle et donc le risque d'échec du receveur. |
| `precision-max-dom` / `-ext` | par équipe | Précision maximale. Sur une passe, une bonne précision envoie la balle sur le coéquipier. Sur une attaque ou un service, elle est **inversée** : la balle s'écarte du joueur adverse visé, qui doit se déplacer davantage. |
| `taux-echec-max-dom` / `-ext` | par équipe | Taux d'échec maximal. Plus il est élevé, plus les joueurs ratent leurs réceptions. |
| `chance-service-smashe-dom` / `-ext` | par équipe | Probabilité (en %) de faire un service smash. Le reste est réparti aléatoirement entre service flottant et service normal. |
| `erreur-max-base` | commun | Angle maximal de décalage de la balle dû au manque de précision. |
| `rayon-perception` | commun | Distance à partir de laquelle un joueur voit la balle et commence à se déplacer vers elle. |
| `rayon-contact` | commun | Distance à partir de laquelle un joueur peut toucher la balle. |
 
À la création, chaque joueur reçoit des valeurs aléatoires bornées par ces maximums. Pour la puissance, la borne basse est fixée pour limiter les écarts entre joueurs.
 
### Probabilité de rater une réception
 
À chaque contact, le receveur rate la balle si un tirage aléatoire entre 0 et 99 est inférieur à :
 
```
(taux-echec + 1.5 × |décalage|) × (vitesse_balle + 0.5)
```
 
Une balle rapide ou mal centrée est donc plus difficile à réceptionner.
 
## Fonctionnement du modèle
 
Joueurs et balle se comportent comme des **automates à états finis** (FSM). Il n'existe pas de variable « état » explicite : les états découlent de l'enchaînement des procédures.
 
### États d'un joueur
 
```
En position ──(balle dans le rayon de perception)──▶ Va vers la balle
                                                        │
                    ┌───(balle dans le rayon de contact)┴(balle hors d'atteinte)───┐
                    ▼                                                              ▼
              Joue la balle ──(échec au test rater)──▶ Raté réception        Raté par manque de temps
                    │                                       │                      │
                    └───────────────────────────────────────┴──────────────────────┴──▶ Revient en position
```
 
| État | Procédure(s) |
|---|---|
| En position | attente sur le patch `pos` |
| Va vers la balle | `move-balle` |
| Joue la balle | `passer`, `rater` |
| Raté réception / raté par manque de temps | `rater`, `move-balle` |
| Revient en position | `revenir-position` |
 
### États de la balle
 
| État | Procédure(s) | Description |
|---|---|---|
| En service | `servir` | Le serveur R1 prend son élan, lance la balle puis frappe vers un receveur adverse. |
| En vol | `move-balle`, `definir-direction` | La balle se dirige vers sa cible, avec un décalage d'angle lié à la précision. |
| Sur un joueur | `passer`, `passer-passeur`, `passer-attaquant`, `passer-adversaire` | Passe vers le passeur R3, puis vers un attaquant (R2 ou R4). Au bout de 2 passes, attaque dans le camp adverse. |
| Au sol | `changement-service` | Attribution du point, rotation si besoin et changement de serveur. |
 
### Actions situées
 
- **Perception locale** : un joueur ne réagit que lorsque la balle entre dans son rayon de perception, et ne peut la jouer que dans son rayon de contact.
- **Décision selon le contexte** : la réussite d'une réception dépend du joueur (taux d'échec), de la balle (vitesse, décalage) et du type de service reçu.
## Résultats
 
Expériences menées avec `simuler-n 100`, en ne faisant varier qu'un paramètre à la fois.
 
| # | Hypothèse | Observation | Verdict |
|---|---|---|---|
| 1 | L'équipe la plus **puissante** gagne. | Avec un gros écart de puissance, l'équipe la **moins** puissante gagne 61 matchs sur 100 (57/100 avec un écart plus faible). La puissance s'applique aussi aux passes entre coéquipiers, qui sont alors plus souvent ratées. | ❌ Infirmée |
| 2 | L'équipe la plus **précise** gagne. | L'équipe précise gagne 65 matchs sur 100 : ses passes sont mieux réussies et ses attaques plus dispersées. L'écart se réduit quand les précisions se rapprochent. | ✅ Validée |
| 3 | L'équipe au **taux d'échec** le plus élevé perd. | Écart de victoires très marqué. C'est le facteur le plus déterminant du modèle. | ✅ Validée |
| 4 | Le **type de service** influe sur la victoire. | Que l'équipe fasse 100 % de smashs ou des services mixtes, les résultats restent proches de 50/50. | ❌ Peu ou pas d'influence |
| 5 | Un **rayon de perception** plus grand allonge les échanges. | La durée moyenne d'un match passe de 1 492 ticks (rayon 10) à 3 209 ticks (rayon 20). | ✅ Validée |
 
## Limites du modèle
 
- La balle retombe toujours dans le terrain : il n'y a ni faute de filet ni balle dehors.
- Une équipe fait toujours le nombre maximal de passes avant d'attaquer.
- La puissance d'un joueur s'applique de la même façon à une passe et à une attaque.
- Le modèle ne simule qu'un seul set.
## Structure du dépôt
 
```
.
├── Projet_Volley_BERGER_BOS.nlogox   # Modèle NetLogo (code + interface)
├── Rapport_Projet.docx               # Rapport du projet
└── README.md
```
 