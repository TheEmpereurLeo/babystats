# BabyStats — Brief pour Claude Code

## C'est quoi ce projet

**BabyStats** est une PWA (Progressive Web App) de suivi de statistiques de **babyfoot** pour un groupe d'amis. L'app est entièrement **self-contained dans un seul fichier HTML** (`BabyStat.html`) — zéro serveur, zéro framework, zéro build step. Elle se déploie sur GitHub Pages et est pensée en priorité pour **iPhone Safari**.

Toute la persistance passe par **`localStorage`** avec la clé `babystats_data` et un système de version (`DB_VERSION = 6`) pour les migrations.

---

## Stack technique

| Élément | Détail |
|---|---|
| Fichier unique | `BabyStat.html` (~2400 lignes) |
| CSS | Variables CSS custom, design system beige/orange, mobile-first |
| JS | Vanilla JS pur, aucun framework |
| Fonts | Google Fonts — Syne (display) + Inter (body) |
| Charts | Chart.js 4.4.0 via CDN |
| Stockage | `localStorage` uniquement |
| Cible | iPhone Safari, responsive desktop avec sidebar |

---

## Architecture du fichier

Le fichier est structuré en 3 blocs dans l'ordre :

1. **`<style>`** — Tout le CSS avec un design system cohérent via CSS variables (palettes beige, orange, red, blue, green, gold, purple)
2. **HTML** — Pages et modals (tout est dans le DOM dès le début, caché via `display:none`)
3. **`<script>`** — Toute la logique JS, organisée en sections commentées avec `═══`

### Navigation

Système de pages custom : chaque page est un `div.page` avec `id="page-XXX"`. La fonction `showPage(name, data)` gère l'activation. Il y a une nav mobile en bas (fixe) et une sidebar desktop (cachée sur mobile).

**Pages existantes :**

| Page | Description |
|---|---|
| `home` | Accueil — hero tournoi actif, trophées, hall of shame |
| `players` | Liste joueurs avec filtres par rôle |
| `player-detail` | Fiche joueur — stats, graphique, historique |
| `teams` | Liste équipes |
| `team-stats` | Stats détaillées d'une équipe |
| `tournaments` | Liste tournois |
| `tournament-detail` | Détail avec tabs (matchs / bracket / équipes / classement) |
| `match` | Page de scoring live avec timer |
| `friendly` | Matchs amicaux |
| `stats` | Classements, graphiques, badges |
| `compare` | Comparaison 1v1 entre joueurs |

---

## Structure des données (localStorage)

```js
db = {
  version: 6,
  players: [{ id, name, role }],          // role: 'attaquant' | 'defenseur' | 'both'
  teams: [{ id, name, attackerId, defenderId, manualWins?, foreverFirst? }],
  tournaments: [{
    id, name, date, status,               // status: 'active' | 'finished'
    teams: [teamId],
    matches: [matchObject],
    matchDuration,                        // minutes
    refereeId,
    tiebreak,                             // 'goal_average' | 'replay'
    phase,                                // 'poule' | 'quart' | 'demi' | 'finale'
    sortiesPerBut,                        // nb de sorties pour déclencher 1 but adverse
    groups: [{ name, teams: [teamId] }],
    knockoutRounds: [],
    specialMatches: {}
  }],
  friendlyMatches: [matchObject]
}
```

**Structure d'un match :**

```js
{
  id, phase, group?,
  teamA, teamB,
  scoreA, scoreB,
  winner: 'a' | 'b' | null,
  finished: boolean,
  finishedAt?,
  sortiesA, sortiesB,           // compteur de "sorties" par équipe
  stats: {
    a: {
      attacker: { id, name, goals, pinches, gamelles, saves, ownGoals },
      defender: { id, name, goals, pinches, gamelles, saves, ownGoals }
    },
    b: { /* même structure */ }
  }
}
```

---

## Vocabulaire métier — CRITIQUE

Ces termes sont spécifiques à cette table de babyfoot. **Ne pas les renommer ni changer leur logique.**

| Terme | Définition |
|---|---|
| **gamelle** | Tir bien cadré qui ne rentre pas (≠ raté). Comptabilisé positivement car ça demande de la précision. |
| **pinch** | Technique : coincer la balle contre le bord pour tirer avec effet |
| **sortie** | Joueur qui sort du terrain. `N` sorties (configurable, défaut 3) = 1 but pour l'adversaire automatiquement |
| **CSC** | Contre-son-camp (own goal) |
| **Gala** | Match entre les équipes éliminées en phase de poule (consolation, pas de qualification) |

> ⚠️ Les **gamelles** ne doivent **pas** être pénalisées dans le score Ballon d'Or. La formule leur donne `+0.5` par gamelle, c'est intentionnel (la table produit beaucoup de gamelles, pénaliser serait injuste).

---

## Fonctionnalités principales

### Tournois

- Génération automatique de poules selon le nombre d'équipes :
  - ≤ 4 équipes → poule unique
  - ≤ 9 équipes → 2 poules
  - \> 9 équipes → poules de 4
- Round-robin dans chaque poule
- Génération automatique du bracket knockout quand toutes les poules sont finies
- Phases : `poule → quart (si ≥8) → demi → petite finale + finale`
- **Match de Gala** : généré automatiquement pour les équipes éliminées en poule
- Tie-break configurable : goal average ou match d'appui

### Scoring live (page `match`)

- Timer configurable avec pause/reset, passe en rouge en overtime
- Counters par joueur et par rôle (attaquant / défenseur)
- Sorties trackées séparément, calcul auto des buts générés
- Score recalculé en temps réel à chaque update de stat

**Formule du score :**
```
scoreA = goals_att_A + goals_def_A + ownGoals_att_B + ownGoals_def_B + floor(sortiesB / sortiesPerBut)
scoreB = goals_att_B + goals_def_B + ownGoals_att_A + ownGoals_def_A + floor(sortiesA / sortiesPerBut)
```

> ⚠️ Ne jamais incrémenter `scoreA/B` directement — toujours le recalculer entièrement depuis les stats brutes.

### Badges / Trophées

**3 badges affichés en home :**

| Badge | Critère |
|---|---|
| 🥇 Ballon d'Or | Score composite : `goals*3 + saves*2 + pinches + wins*4 + gamelles*0.5 - ownGoals` |
| 👟 Soulier d'Or | Meilleur buteur (goals) |
| 🛡 Master of Defence | Plus d'arrêts (saves) |

**2 badges supplémentaires dans Stats > Badges :**

| Badge | Critère |
|---|---|
| 🤏 Master of Pinch | Plus de pinches |
| 🤦 Roi du CSC | Plus de buts contre son camp |

### Badge permanent spécial

👑 **"À jamais les 1ers"** — badge violet attribué à l'équipe contenant les joueurs nommés `Isayan` ou `Léo` (détection par nom dans `isForeverFirstTeam()`), ou manuellement via `team.foreverFirst = true`.

C'est un easter egg commémorant les premiers vainqueurs du tournoi. **Ne pas supprimer cette logique.**

---

## Points techniques importants

### Variables globales critiques

| Variable | Rôle |
|---|---|
| `currentTournament` | Pointe sur le tournoi affiché — mis à jour dans `renderTournamentDetail()` |
| `isFriendlyMatch` | Boolean — détermine si le match en cours est amical ou tournoi |
| `currentSortiesPerBut` | Règle sorties active pour le match en cours |
| `currentSortiesA/B` | Compteurs de sorties du match en cours |
| `chartInstances` | Stocke les instances Chart.js pour les détruire avant recréation |

### Règles à respecter

- **Charts** : toujours appeler `destroyChart(id)` avant de recréer un graphique — sinon memory leak et canvas corrompu
- **Migration DB** : dans `dbLoad()`, des corrections sont appliquées sur les anciens saves. Si tu ajoutes un nouveau champ à un objet existant, l'ajouter ici aussi avec une valeur par défaut
- **Swap de rôles** : possible pendant ou après un match via `modal-swap-roles`. Les stats restent attachées aux joueurs, seul l'affichage attaquant/défenseur s'inverse
- **`getMatchList()`** : retourne la bonne liste de matchs selon `isFriendlyMatch` — toujours passer par cette fonction pour accéder au match en cours

### Design system CSS

Toutes les couleurs sont des variables CSS dans `:root`. Ne pas hardcoder de couleurs hex dans le JS ou dans du HTML inline — utiliser les variables existantes (`var(--orange-400)`, etc.).

Palettes disponibles : `beige`, `orange`, `red`, `blue`, `green`, `gold`, `purple`

---

## Ce qui est incomplet / connu

- Le `tiebreak: 'replay'` est configurable mais le match d'appui n'est **pas encore auto-généré** (à implémenter)
- La page `compare` n'est **pas dans la nav mobile** (accessible uniquement en sidebar desktop)
- Pas de PWA manifest ni service worker (juste les meta tags apple-mobile-web-app)
