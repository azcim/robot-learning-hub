# Robot Learning Hub

Appunti su robot learning — paradigmi di apprendimento, modelli, dati e infrastruttura — pubblicati come sito navigabile con [Quartz](https://quartz.jzhao.xyz/), incluse mappa concettuale (canvas), grafo dei collegamenti e ricerca full-text.

🔗 Sito pubblicato: https://azcim.github.io/robot-learning-hub

## Struttura

Le note vivono in [`content/`](content), che è anche il vault Obsidian: apri quella cartella in Obsidian per continuare a scrivere.

- `content/index.md` — homepage
- `content/Griglie decisionali.md`, `content/Percorsi end-to-end.md` — confronti e pipeline
- `content/Robot Learning.canvas` — mappa concettuale, resa navigabile sul sito
- `content/Approfondimenti/` — paradigmi e modelli (IL, RL, ACT, Diffusion Policy, VLA, WAM...)
- `content/Infrastruttura/` — framework, hardware, simulazione

## Sviluppo locale

```bash
npm ci
npx quartz build --serve
```

Il sito è servito su `http://localhost:8080` e si ricarica ad ogni modifica in `content/`.

## Deploy

Il push su `main` innesca il workflow [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml), che builda il sito con Quartz e lo pubblica su GitHub Pages.
