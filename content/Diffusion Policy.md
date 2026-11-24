---
tags:
---





TODO






# Diffusion Policy

*Addestrata con [[Imitation Learning]] · rifinibile con RL via [[DPPO]]*

## Come funziona

Il chunk di azioni viene generato partendo da **rumore gaussiano puro** e ripulito iterativamente in più passi di denoising, condizionati dalle osservazioni visive.

Rappresentando distribuzioni **multi-picco**, impara senza sforzo traiettorie alternative per lo stesso compito — è la risposta più diretta al problema della [[Imitation Learning#3. Multimodalità|multimodalità]].

## Varianti

- **CNN 1D** — più semplice e robusta, il default
- **Transformer** — scala meglio con i dati
- **DP3** — usa **nuvole di punti** invece dei pixel: robustezza alle variazioni di illuminazione

## Limiti

- **inferenza più lenta**: molti passi di denoising per chunk
- resta **task-specific**
- il costo di denoising si riduce con campionatori tipo DDIM (vedi [[DPPO]], che lo comprime a ~5 step)

## Requisiti

- **50-200 dimostrazioni** per task
- ~24 GB di VRAM in training
- formato dati tipico: **Zarr** (array piatti compressi + `episode_ends`) → [[Dati#Formati di archiviazione]]

---

## Collegato

[[ACT]] · [[DPPO]] · [[Imitation Learning]] · [[Performance in esecuzione]]
