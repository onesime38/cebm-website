# CEBM — Communauté des Étudiants Burundais au Maroc

Site public et espace d'administration de la CEBM. Les visiteurs y consultent
les événements, les annonces, la galerie, les contacts et les guides ; le
bureau et les représentants de ville publient ce contenu.

> **En cours de refonte.** Aujourd'hui, l'application tient dans
> `src/App.tsx` et stocke les données dans le navigateur. La cible est une API
> Vercel qui lit et écrit dans un Google Sheet et un dossier Google Drive
> appartenant à l'association. Voir [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Prérequis

- Node.js 22 (voir `.nvmrc`)
- npm

## Démarrer

```bash
npm install
npm run dev          # front-end seul sur http://localhost:5173
```

Quand l'API existera (phase 1), le site complet se lancera avec
`vercel dev` sur http://localhost:3000. Il faudra alors `vercel login`,
`vercel link` et un fichier `.env.local` copié depuis `.env.example`
(valeurs : [`docs/GOOGLE_SETUP.md`](docs/GOOGLE_SETUP.md)).

## Commandes

| Commande | Rôle |
|---|---|
| `npm run dev` | serveur de développement Vite |
| `npm run build` | vérification TypeScript + build de production dans `dist/` |
| `npm run lint` | ESLint |
| `npm run typecheck` | vérification TypeScript seule |
| `npm test` | tests unitaires (Vitest) |
| `npm run preview` | servir le build localement |

`npm run lint`, `npm test` et `npm run build` doivent passer avant chaque commit.

## Documentation

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — architecture, données, API
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — phases et tâches
- [`docs/SECURITY.md`](docs/SECURITY.md) — règles et checklist de sécurité
- [`docs/GOOGLE_SETUP.md`](docs/GOOGLE_SETUP.md) — configuration Google et Vercel
- [`docs/WORKFLOW.md`](docs/WORKFLOW.md) — travailler avec Claude Code
- [`CLAUDE.md`](CLAUDE.md) — conventions du projet

## Contribuer

Une tâche de la roadmap = une branche (`phase-0/cleanup`, …) = une pull
request relue par l'autre développeur. Commits en
[Conventional Commits](https://www.conventionalcommits.org/fr/), message en
français. Ne jamais committer de fichier `.env*` (sauf `.env.example`).
