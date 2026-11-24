---
tags:
---
# Imitation Learning

**Paradigma di addestramento supervisionato:** Si parte da un dataset di traiettorie raccolte da un esperto (sequenze di stati e azioni, tipicamente via teleoperazione) e si cerca la policy che riproduce l'azione dell'esperto nello stesso stato.

**Behavioral Cloning (BC)** è la forma base: una regressione che minimizza l'errore quadratico fra azione dimostrata e azione predetta.

$$  
\min_{\theta} \frac{1}{|\mathcal{D}|} \sum_{(\mathbf{s},\mathbf{a}) \in \mathcal{D}} \lVert \mathbf{a} - \pi_\theta(\mathbf{s}) \rVert^2  
$$

**È un paradigma, non un'architettura:** Dice da dove viene il segnale di apprendimento, non che forma ha la policy.
- Lo stesso modello può essere addestrato con paradigmi diversi in momenti diversi: una [[Diffusion Policy]] nasce con **IL** e si rifinisce con **RL** ([[DPPO]]); un [[VLA]] fa pre-training self-supervised, fine-tuning con **IL** e post-training con **RL**.

---

## I tre modi in cui il BC fallisce

### 1. Compounding error

**Il problema:** gli stati visitati dalla policy non sono quelli visitati dall'esperto (_distribution shift_, o _covariate shift_).
- Nel supervised learning classico vale l'assunzione **i.i.d.**: classificare male l'immagine A di un gatto non cambia il contenuto dell'immagine B, quindi gli errori restano isolati.
- Nel BC l'assunzione **non vale**, perché le azioni scelte dal robot determinano gli stati che osserverà dopo. La policy parte da $s_1$, produce un'azione con un piccolo errore inevitabile, finisce in uno stato vicino a $s_2$ ma non identico, e siccome quello stato non era nei dati la predizione successiva è ancora più sbagliata.
- Lo shift può nascere anche da cause del tutto esterne al robot: un cambio di illuminazione, una telecamera urtata di due millimetri, uno sfondo diverso.

#### Soluzioni

>**Action chunking:** predire un blocco di $k$ azioni future invece di un passo alla volta, per esempio un secondo intero di movimento. 
- Se si decide una volta ogni $k$ passi, il numero di decisioni lungo il compito si riduce di un fattore $k$, e con esso le occasioni di accumulare errore.
- C'è un secondo beneficio meno ovvio: il chunk **assorbe le pause e le esitazioni** dell'operatore. Una policy passo-per-passo che incontra un momento in cui l'umano si era fermato a pensare tende a bloccarsi o a entrare in un ciclo di incertezza; un blocco di azioni passa oltre.
- Il prezzo è che eseguire un chunk rigidamente in open loop rende il movimento scattoso a ogni cambio di blocco e cieco a quello che succede nel frattempo. Le due risposte sono il _temporal ensembling_ di ACT e il _receding horizon control_ di Diffusion Policy.

>**Data augmentation:** creare artificialmente gli esempi di recupero che mancano.
- L'esempio canonico è la guida autonoma end-to-end di NVIDIA (2016). Il sistema funzionava male perché quasi tutti i dati mostravano la guida al centro corsia: appena l'auto derivava verso il bordo si trovava in uno stato mai visto e non sapeva rientrare.
- La soluzione fu montare **tre telecamere** invece di una e etichettare sistematicamente i fotogrammi della camera sinistra con "sterza a destra" e quelli della destra con "sterza a sinistra".
- Il punto elegante: si ottengono esempi espliciti di auto-correzione **senza che nessun guidatore debba mai far sbandare davvero l'auto**. Sono dati finti che insegnano una relazione vera.

>**DAgger** (_Dataset Aggregation_, Ross et al. 2011): far convergere la distribuzione degli stati di addestramento verso quella che la policy incontra davvero.
- Si allena una prima policy sulle dimostrazioni iniziali e successivamente si fa eseguire il compito alla policy **in totale autonomia**, registrando gli stati visitati (derive comprese).
- Un esperto umano (o un oracolo) indica **a posteriori** l'azione corretta $\pi^*(s)$ per quegli stati, e queste nuove coppie di dati si aggregano ai dati raccolti precedentemente e si riallena, iterando il processo.
- **Risultato teorico:** l'errore torna a crescere **linearmente**, $\mathcal{O}(T\epsilon)$, con un numero di iterazioni che scala quasi linearmente con l'orizzonte.
- **Il costo è il collo di bottiglia:** Serve l'esperto disponibile a ogni iterazione, e per molti compiti di manipolazione l'etichettatura a posteriori è concettualmente difficile, perché a un umano che guarda un fotogramma fuori contesto non è ovvio quale fosse l'azione giusta in quel preciso istante.

>**Human-Gated DAgger** (_online intervention_): Il robot opera sotto supervisione in tempo reale e l'operatore **prende il controllo solo quando serve**, fornendo una dimostrazione parziale per il resto del compito. Appena il sistema è di nuovo in sicurezza il controllo torna al robot.
- **Dati con errori e recuperi:** inserire di proposito nel dataset episodi in cui l'operatore sbaglia e poi corregge, o parte da pose iniziali anomale.
    - È il punto più controintuitivo di tutta la pagina: **il BC funziona meglio se i dati contengono errori**, purché seguiti dalla correzione giusta.
- Molto più economico del DAgger classico, ma sposta il problema anziché eliminarlo: **come si rileva che serve intervenire?** Per evitare una supervisione costante servirebbe che il robot riconoscesse da solo la propria incertezza e chiedesse aiuto prima di sbagliare, il che è ancora un problema aperto.

---

### 2. Comportamento non markoviano

>**Il problema:** l'umano decide su contesto e storia, mentre la formulazione standard della policy assume che l'azione dipenda **solo dall'osservazione corrente**.
- Davanti alla stessa identica scena una persona può agire in modo diverso in momenti diversi, perché si basa sulla storia recente, su un'intenzione formulata dieci secondi prima, perfino sul proprio stato di attenzione.
- Nella manipolazione fine questo si manifesta come pause, esitazioni e piccoli riallineamenti che non sono spiegabili dal solo fotogramma attuale.

>**Causal confusion:** Dare in input gli ultimi $N$ fotogrammi con un'architettura sequenziale **non sempre migliora e spesso peggiora**. 
- Avendo accesso alla storia, il modello scopre che la propria azione passata è un predittore eccellente dell'azione presente, e impara semplicemente a copiarla. 
- La loss scende, perché quella correlazione nei dati è reale, ma il modello non ha imparato il compito e in esecuzione la scorciatoia non regge.

>**L'osservazione privilegiata** è il caso specifico che pesa di più in teleoperazione, ed è facile trascurarlo quando si progetta il setup di raccolta.
- Chi teleopera **vede la scena con i propri occhi**, da una posizione favorevole e con visione binoculare. La policy imparerà invece a vedere solo ciò che inquadrano le telecamere di bordo, con campo visivo limitato, occlusioni e nessuna profondità diretta.
- L'esperto compie quindi azioni basate su informazioni che il robot **non avrà mai**. Dal punto di vista della policy quelle azioni appaiono arbitrarie, perché la causa che le ha generate non è nei dati.

**_Mitigazione:_** far teleoperare guardando gli stessi flussi video del robot invece della scena reale, e aggiungere telecamere da polso che riducano il divario di prospettiva.

---

### 3. Multimodalità

>**Il problema:** se metà delle demo aggira l'ostacolo a sinistra, metà a destra, e l'MSE impara la media, cioè andare dritto contro l'ostacolo. 
- Una policy che predice un singolo valore infatti assume implicitamente che la distribuzione delle azioni corrette sia una gaussiana unimodale, e quando non lo è produce un risultato che sta **fra i picchi**, dove la densità di probabilità è minima.
- L'azione risultante è quindi una che **nessun esperto ha mai eseguito**, ed è spesso la peggiore possibile.
- È l'unico dei tre fallimenti che non dipende dai dati ma dalla **forma della policy**: si verifica anche con dati perfetti.

>**Nasce da due cause distinte**, entrambe inevitabili in pratica:
- **Giunti ridondanti:** un robot con ridondanza cinematica ha infinite configurazioni articolari che raggiungono la stessa posa dell'end-effector, perciò lo stesso obiettivo ammette molte soluzioni ugualmente valide.
- **Expert inconsistency:** operatori diversi eseguono lo stesso compito in modi diversi, e anche lo stesso operatore cambia strategia fra una prova e l'altra.

#### Soluzioni

Tutte le soluzioni moderne hanno come obiettivo rendere la policy capace di rappresentare un'intera **distribuzione di probabilità** condizionata all'osservazione, dalla quale campionare una singola azione valida.

>**Mixture Density Networks (MDN):** la rete predice i parametri (medie, varianze e pesi) di una combinazione lineare di \(K\) distribuzioni gaussiane, permettendo di modellare più picchi contemporaneamente.
- È molto più espressiva di una singola gaussiana, ma richiede di **fissare a priori il numero di modi \(K\)** come iperparametro. Poiché nella realtà il numero di strategie valide cambia continuamente a seconda dello stato, questo vincolo rigido limita la flessibilità del modello proprio nei contesti più complessi.

>**Conditional Variational Autoencoders (CVAE):** introduce una variabile latente \($z$\) che cattura lo "stile" o l'intenzione specifica dell'azione, trasformando la policy in una funzione deterministica condizionata: $\pi(\mathbf{a}\mid \mathbf{o}, z) = f_\theta(\mathbf{o}, z)$
- Permette di generare traiettorie diverse ma internamente coerenti a partire dalla stessa osservazione, evitando qualsiasi effetto di media. 
- La sfida principale riguarda **come campionare o scegliere \(z\) in fase di inferenza (test time)** per garantire azioni sicure e in-distribuzione (vedi [[ACT]]).

>**Diffusion Policies (Modelli a Diffusione):** anziché mappare direttamente l'osservazione nell'azione, la rete impara un processo iterativo di _denoising_. 
- Partendo da puro rumore gaussiano, l'azione viene raffinata passo dopo passo attraverso una serie di piccoli aggiornamenti condizionati sull'osservazione.
- Rappresenta in modo nativo distribuzioni multimodali arbitrarie senza doverne specificare il numero di picchi, garantendo al contempo un addestramento molto stabile. Il principale svantaggio è l'**alto costo computazionale in inferenza**, poiché generare una singola azione richiede decine di valutazioni sequenziali della rete.

>**Flow Matching:** rappresenta l'evoluzione diretta dei modelli a diffusione ed è oggi l'approccio dominante nei modelli robotici generalisti (come i VLA).
- Invece di seguire le traiettorie stocastiche e curve tipiche della diffusione, il modello apprende un campo vettoriale che trasporta la distribuzione del rumore verso quella delle azioni lungo **percorsi quasi rettilinei** (governati da un'equazione differenziale ordinaria, ODE).
- La linearità delle traiettorie permette di integrare il flusso con pochissimi passi senza perdere precisione, risultando molto più veloce e stabile da addestrare rispetto alle soluzioni precedenti, motivo per cui costituisce il cuore dell'_action expert_ in modelli allo stato dell'arte.

---

## Quando usarlo

- È il **default per la manipolazione**: compiti come "ripiegare una maglietta" o "inserire un connettore" sono difficilissimi da formalizzare in una funzione di reward, mentre sono facili da _mostrare_.
- La regola pratica sugli umanoidi:
    - **Manipolazione (bassa frequenza) → IL**, perché l'interazione con oggetti è impossibile da descrivere in una reward
    - **Locomozione e controllo corporeo (alta frequenza) → RL in simulazione**, perché la fisica rigida è simulata bene e le reward sono chiare
- In termini di deploy è il paradigma **più veloce da portare a risultato**, per ragioni strutturali:
    - Niente simulatore obbligatorio
    - Niente reward da progettare, che è il vero collo di bottiglia dell'RL
    - Niente problema di sim-to-real, perché non c'è stata simulazione
	- Si addestra in **poche ore su una singola GPU consumer con 50-200 dimostrazioni** per task.
- **Quando non usarlo:**
    - quando il compito è facile da specificare e difficile da mostrare, tipicamente l'equilibrio dinamico
    - quando serve superare la qualità dell'esperto (dove va quantomeno affiancato da RL)
    - quando servono molti compiti diversi, dove conviene un [[VLA]] da fine-tunare invece di una policy per compito

---

## Che dati servono

Episodi di teleoperazione, ciascuno con la triade **sincronizzata**:
- **Percezione visiva:** i flussi delle camere di bordo
- **Propriocezione:** angoli dei giunti, posa dell'end-effector
- **Comandi motore:** le azioni effettivamente inviate
- **Ordine di grandezza:** 50-200 demo per un task ACT o Diffusion Policy ben delimitato. ACT raggiunge il 96% su un inserimento di precisione con 50 dimostrazioni.

>La capacità di generalizzare segue approssimativamente una **legge di potenza** rispetto al numero di **ambienti** e di **oggetti**. La **diversità di ambienti e oggetti conta molto più del numero assoluto di dimostrazioni**: superata una certa soglia per ambiente, aggiungerne altre ha effetto minimo.

- **Tre requisiti di qualità non negoziabili**, perché **il modello replica fedelmente le imperfezioni del dataset**:
    - **Sincronizzazione temporale** fra frame, stato dei giunti e comandi. Un disallineamento sistematico insegna al modello una relazione causale sbagliata, e non si vede finché il robot non si muove.
    - **Frequenza costante**, e identica fra campionamento in training e inferenza della policy. Se non coincidono, il movimento risulta rallentato o accelerato rispetto alla dimostrazione anche con tutto il resto corretto.
    - **Gestione dei fallimenti:** inserire di proposito episodi in cui l'operatore corregge traiettorie errate o parte da pose anomale. È la contromisura più efficace al compounding error, più di qualunque accorgimento architetturale.

>**Corollario sulla velocità:** la velocità del gesto si imposta **in raccolta dati**, teleoperando più rapidamente, non solo alzando la frequenza di esecuzione.

---