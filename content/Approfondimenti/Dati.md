---
tags:
---
Il collo di bottiglia del robot learning sono i dati. Un modello visione-linguaggio impara da miliardi di immagini e di testi presi dal web. Per i robot non esiste niente di equivalente, perché ogni dimostrazione con le azioni registrate costa il tempo di un operatore e l'uso di un robot. Quasi tutto ciò che si fa oggi sui dati è un modo per ridurre quante dimostrazioni servono sul robot vero, sostituendole con qualcosa di più economico.

### La piramide dei dati

L'immagine che riassume la strategia è la piramide dei dati di NVIDIA, introdotta con GR00T N1. In basso ci sono i dati più abbondanti e meno specifici: i video di internet, che hanno la semantica ma né il punto di vista giusto né le azioni. Poi il video egocentrico, ripreso da una camera indossata: ha il punto di vista giusto, ma ancora nessuna azione. Poi l'egocentrico arricchito, dove oltre al video si ha la posa tridimensionale di mani e testa. Poi la cattura con gripper portatile, in cui una persona muove uno strumento che registra traiettorie quasi identiche a quelle di un robot. Sopra stanno simulazione e teleoperazione, e in cima il deployment: i dati prodotti dal robot mentre lavora davvero.

---

## La piramide dei dati (NVIDIA, GR00T N1)

In basso ci sono i dati più abbondanti e meno specifici: i video di internet, che hanno la semantica ma né il punto di vista giusto né le azioni. Poi il video egocentrico, ripreso da una camera indossata: ha il punto di vista giusto, ma ancora nessuna azione. Poi l'egocentrico arricchito, dove oltre al video si ha la posa tridimensionale di mani e testa. Poi la cattura con gripper portatile, in cui una persona muove uno strumento che registra traiettorie quasi identiche a quelle di un robot. Sopra stanno simulazione e teleoperazione, e in cima il deployment: i dati prodotti dal robot mentre lavora davvero.

| Livello                             | Che cosa è                               | Punto di vista | Azioni                 | Costo     |
| ----------------------------------- | ---------------------------------------- | -------------- | ---------------------- | --------- |
| 1. **Video di internet**            | YouTube, tutorial, film                  | sbagliato      | nessuna                | basso     |
| 2. **Video egocentrico**            | camera indossata (Ego4D)                 | **giusto**     | nessuna                | basso     |
| 3. **Egocentrico arricchito**       | + posa 3D di mani e testa (EgoDex)       | giusto         | implicite nella posa   | basso     |
| 4. **Gripper portatile / guanti**   | UMI, gripper handheld, guanti tipo Manus | giusto         | **quasi robotiche**    | medio     |
| 5. **Simulazione e dati sintetici** | Isaac Lab, SynGrasp-1B, ArtVIP           | giusto         | **vere** (ma simulate) | medio     |
| 6. **Teleoperazione**               | operatore che guida il robot vero        | **giusto**     | **vere**               | alto      |
| 7. **Deployment**                   | il robot mentre lavora davvero           | giusto         | vere                   | altissimo |

---

### Come si usano i dati senza azioni

Il problema pratico è che il video di una persona che apre un cassetto dice che cosa succede, non quali comandi mandare ai motori. Ci sono quattro modi per colmare il vuoto, dal più astratto al più diretto.

- **Azioni Latenti:** Un modello, di solito un VQ-VAE, impara a codificare ciò che cambia tra due fotogrammi in un piccolo codice discreto, e quei codici fanno da pseudo-azioni su cui pre-addestrare il VLA. Lo usano GR00T N1 e GO-1 di AgiBot.
- **Inverse Dynamics Models (IDM):** Un modello addestrato su dati robotici reali stima, dati due fotogrammi, l'azione che ha portato dall'uno all'altro. Si può applicare a video generati o a video senza log. È il metodo con cui GR00T N1 ha trasformato 88 ore di teleoperazione in 827 ore di «traiettorie neurali», usando video prodotti da modelli generativi.
- **Retargeting Cinematico:** Se il video contiene la posa 3D delle mani umane, questa viene mappata sulla cinematica della mano robotica tramite ottimizzazione geometrica.
- **Spazi d'Azione Condivisi:** Si definisce uno spazio di controllo comune, calcolabile sia sull'umano sia sul robot, per esempio il movimento relativo del polso. Allora l'umano diventa un embodiment come un altro, con azioni vere. È la strada di GR00T N1.7, pre-addestrato anche su circa 20.000 ore di video egocentrici umani, il dataset EgoScale. 

#### Il quinto meccanismo: non estrarre nessuna azione

> [!note] È la mossa dei world model
> I quattro sopra condividono un presupposto: per usare un video bisogna prima **ricavarne un'etichetta d'azione**, esplicita o latente. I [[WAM e World Models|WFM/WAM]] rovesciano il problema: l'obiettivo non è predire un'azione ma **predire i token video futuri**. La dinamica non viene estratta, resta **implicita nei pesi e nelle feature**.

Nessuna perdita di informazione nel passaggio video → azione, e nessun rischio di codificare rumore visivo. Le azioni robotiche servono solo alla fine, per **allineare** i comandi motori a una fisica già compresa, e ne servono molte meno.

*Prezzo:* pre-training enorme (modelli fino a 14B su milioni di video) e runtime pesante.

---
### **Dataset di Riferimento**

- **Bracci Singoli (Generalisti):**
    - **Open X-Embodiment (OXE):** Oltre 1 milione di traiettorie su 22 robot in formato RLDS. Estremamente vario, ma eterogeneo per frequenze e sensori.
    - **DROID:** Circa 76.000 traiettorie concentrate su un unico braccio (Franka) in 564 ambienti differenti. Ideale per benchmark di fine-tuning.
- **Umanoidi e Robot Bi-braccio:**
    - **RoboMIND (X-Humanoid):** Dataset standardizzato con protocollo uniforme. La v2.0 supera le 310.000 traiettorie bi-braccio su 6 embodiment, inclusi dati sul robot bipede Tien Kung (19.000 episodi), simulazioni e annotazioni dei fallimenti.
    - **AgiBot World:** La più grande raccolta per umanoidi bi-braccio, con oltre 1 milione di traiettorie dichiarate.
- **Dati Sintetici e Simulati:**
    - **SynGrasp-1B (Galbot):** 1 miliardo di fotogrammi sintetici focalizzati sul grasping, usati per addestrare GraspVLA.
    - **ArtVIP (X-Humanoid):** Libreria di oggetti articolati ad alta fedeltà per ambienti simulati.
- **Video Umani:**
    - **Ego4D / EgoDex (Apple):** Rispettivamente 3.670 ore di video in prima persona e 829 ore dotate di tracking 3D delle dita per il retargeting.

---

### **Formati di Archiviazione dei Dati**

Tutti i formati contengono stati, azioni, flussi video, metadati di frequenza e statistiche di normalizzazione, ma differiscono nell'architettura di storage:

- **MCAP / Rosbag:** Log grezzi del robot. Registra ogni messaggio con il suo timestamp e non perde nulla, ma non è pronto per l'addestramento, perché i flussi sono asincroni.
- **HDF5:** Struttura gerarchica per singolo episodio (usato da ACT, ALOHA, Isaac Lab). Semplice da implementare ma inefficiente su scala massiva.
- **Zarr:** Ottimizzato per letture casuali parallele (usato da Diffusion Policy e UMI). Concatena tutti gli episodi in array piatti compressi e usa un array ausiliario (`episode_ends`) per i confini.
- RLDS è il formato di Open X-Embodiment: episodi fatti di passi, sopra TensorFlow Datasets, e di fatto legato a TensorFlow.
- **LeRobot (Hugging Face):** Sta diventando lo standard di fatto per l'ecosistema aperto. La **versione 3** supporta dataset massivi in streaming memorizzando file Parquet per metadati/stati e video MP4 divisi in chunk pronti per l'Hub.

---

### **Requisiti di qualità dei dati**

Un modello di Imitation Learning replica fedelmente le imperfezioni del dataset. Per mitigare i fallimenti sul robot reale, la pipeline di acquisizione deve garantire:

- **Sincronizzazione temporale:** Allineamento millimetrico tra i frame delle telecamere, lo stato dei giunti e i comandi inviati.
- **Frequenza costante:** Il campionamento in addestramento deve coincidere esattamente con la frequenza di inferenza della policy in esecuzione.
- **Gestione dei fallimenti:** Inserire intenzionalmente nel dataset episodi in cui l'operatore corregge traiettorie errate o posizioni di partenza anomale, insegnando al modello come recuperare dagli errori (fondamentale contro l'errore composto).
