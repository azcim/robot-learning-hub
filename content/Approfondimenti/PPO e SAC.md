---
tags:
---






TODO








# PPO e SAC

*I due algoritmi [[Reinforcement Learning|RL]] che contano in robotica. Entrambi model-free e policy-based con critic di supporto; rispondono in modo opposto al limite dell'on-policy.*

---

## Prima: perché actor-critic

I metodi puri hanno debolezze speculari:

- **policy-based puri** (REINFORCE) → gradienti non distorti ma ad **altissima varianza**
- **value-based puri** ([[Q-learning e DQN]]) → varianza bassa grazie al bootstrapping ma **solo azioni discrete**

**Actor-Critic** unisce i due mondi: l'**Actor** è la policy e sceglie le azioni, il **Critic** è una value function e valuta. Letto come *Generalized Policy Iteration*, il Critic fa *policy evaluation* e l'Actor *policy improvement* — ma senza conoscere il modello dell'ambiente.

---

## PPO — Proximal Policy Optimization (on-policy)

- **Obiettivo:** evitare che un aggiornamento troppo aggressivo distrugga la stabilità della policy.
- **Come:** riusa lo stesso blocco di dati per pochi passi di ottimizzazione, con un meccanismo di **clipping** che azzera il gradiente se la nuova policy si allontana troppo da quella che ha raccolto i dati.
- La loss bilancia tre componenti: actor (policy protetta), critic (valore dello stato), entropia (esplorazione).
- **Pro:** semplice e stabile, ideale per addestramenti massivi in simulazione.
- **Contro:** bassa efficienza campionaria — i dati sono usa e getta.

## SAC — Soft Actor-Critic (off-policy)

- **Obiettivo:** l'efficienza campionaria.
- **Come:** **experience replay buffer** (come DQN): ogni transizione viene riutilizzata migliaia di volte. Massimizza reward **ed entropia** della policy, così il comportamento resta vario e non collassa prematuramente; doppio critic per evitare sovrastime ottimistiche.
- **Pro:** straordinaria efficienza campionaria, nativo per azioni continue — l'opzione naturale quando i dati si raccolgono **sul robot vero**.
- **Contro:** architettura complessa (allena cinque reti) e molto sensibile agli iperparametri.

---

## Quando

- **Da zero:** locomozione ed equilibrio. → [[Reinforcement Learning#Quando usarlo]]
- **Dopo IL:** post-training su manipolazione. → [[DPPO]]
- **Navigazione:** di solito batte un planner classico solo su terreno irregolare, folla o vincoli dinamici.

## Hardware

- **Training:** il carico più pesante del campo. Isaac Lab fa girare migliaia di ambienti in parallelo → **48 GB di VRAM**.
- **Inferenza:** una policy di locomozione è una MLP piccola, gira ovunque — tipicamente **sul computer di bordo**, perché deve stare in un loop ad alta frequenza senza latenze di rete.

---

## Collegato

[[Reinforcement Learning]] · [[Q-learning e DQN]] · [[DPPO]] · [[Simulazione]] · [[Hardware]]
