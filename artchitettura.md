# Architettura del progetto ZebraRestorer

## Scopo del progetto
`ZebraRestorer` e' un progetto di ricostruzione e analisi di immagini bio-mediche di larve di zebrafish acquisite in condizioni di bassa luce. La logica reale del progetto vive quasi interamente nel notebook `pescetti_unet_notebook_cells.ipynb`; il `README.md` descrive l'idea generale, ma il notebook contiene la pipeline effettivamente eseguita.

L'obiettivo principale e' trasformare dati grezzi molto rumorosi in:

1. una ricostruzione intensita' -> `AS`-like tramite una `U-Net`;
2. una stima dei bordi anatomici tramite reti secondarie oppure filtri classici;
3. metriche quantitative che confrontano input, ricostruzione e bordi predetti.

## Dove sta la logica
- `README.md`: panoramica concettuale.
- `pescetti_unet_notebook_cells.ipynb`: implementazione completa della pipeline.
- `Pescetto1PCg.asc`, `Pescetto1BSg.asc`, `Pescetto1AS.asc`: dati di input principali.
- `outputs/checkpoints/`: checkpoint dei modelli addestrati.
- `outputs/edgemetrics/`: immagini e debug output del ramo edge.

## Idea generale in una frase
Il progetto prende misure grezze riga-per-riga, le ricompone in immagini 2D, applica un preprocessing statistico, allena una `U-Net` per ricostruire un'immagine pulita simile al target `AS`, poi opzionalmente usa altre reti o metodi classici per estrarre i bordi.

## Tipi di dato usati nel progetto

### 1. `AS`
E' il riferimento anatomico/ground truth 2D. Viene letto direttamente come immagine finale `H x W`.

### 2. `PC`
E' il segnale di input principale nel notebook di base. Non nasce come singola immagine 2D finale, ma come molte righe temporali che devono essere raggruppate.

### 3. `BS`
E' un secondo segnale di input alternativo, trattato in modo analogo a `PC`.

## Sequenza temporale completa della pipeline

## Fase 0: configurazione sperimentale
La pipeline parte dalla `Config` nel notebook, che definisce:

- file da leggere;
- `stack_size_input = 1000`;
- `stack_mode = "sum"` oppure `"mean"`;
- `input_mode = "pc"`, `"bs"` oppure `"pcbs"`;
- dimensione patch e stride;
- epoche, learning rate, loss, device.

Questa fase non elabora ancora immagini, ma decide **come** saranno costruite e trattate.

## Fase 1: lettura dei file `.asc`
Classe coinvolta: `PescettiDataLoader`.

Per ogni file:

1. viene aperto il file testuale;
2. ogni riga viene letta con `csv.reader`;
3. vengono scartate le celle vuote;
4. viene scartata la prima colonna, trattata come metadato;
5. il resto della riga viene convertito in `float32`;
6. tutte le righe vengono impilate in una matrice 2D.

Output di questa fase:

- `as_img`: immagine target finale gia' 2D;
- `pc_rows`: matrice con moltissime righe temporali;
- `bs_rows`: stessa idea per il secondo canale.

Questa e' una fase **classica**, senza rete neurale e senza filtri di immagine nel senso stretto.

## Fase 2: ricostruzione dell'immagine 2D da dati temporali
Funzione coinvolta: `_stack_input_rows`.

Qui avviene uno dei passaggi piu' importanti del progetto.

`PC` e `BS` non vengono usati subito come immagini. Prima il codice controlla che il numero di righe di input sia compatibile con il target `AS`, secondo il rapporto:

`righe_input = righe_target * stack_size_input`

Poi i dati vengono riorganizzati con un `reshape` nel formato:

`(numero_frame_temporali, numero_righe, numero_pixel)`

Nel notebook il caso previsto e':

- `1000` frame temporali;
- `H` righe;
- `W` pixel per riga.

Dopo il reshape, avviene il collasso temporale:

- con `stack_mode = "sum"`: somma dei `1000` frame;
- con `stack_mode = "mean"`: media dei `1000` frame.

Output di questa fase:

- `pc_img`: immagine 2D `H x W`;
- `bs_img`: immagine 2D `H x W`.

Questa fase e' **classica**: non c'e' alcuna NN, ma c'e' una trasformazione strutturale fondamentale del dato.

## Fase 3: scelta dell'immagine che entra nella pipeline principale
Il notebook decide quale immagine usare come input della ricostruzione:

- `input_mode = "pc"` -> usa `pc_img`;
- `input_mode = "bs"` -> usa `bs_img`;
- `input_mode = "pcbs"` -> usa `0.5 * (pc_img + bs_img)`.

Il target resta sempre:

- `y_raw = as_img`

Quindi a questo punto abbiamo:

- `x_raw`: immagine rumorosa da restaurare;
- `y_raw`: immagine target anatomica.

Anche questa e' una fase **classica**.

## Fase 4: preprocessing statistico e normalizzazione
Classe coinvolta: `PreProcessor`.

Questa e' la fase in cui l'immagine viene trasformata prima di essere data alla rete.

### 4.1 Trasformata di Anscombe sull'input
Sull'input `x_raw` viene applicata:

`2 * sqrt(x + 3/8)`

Il codice prima forza i valori negativi a `0`.

Scopo:

- stabilizzare la varianza;
- rendere piu' trattabile il rumore di tipo Poisson;
- presentare alla rete un input piu' regolare.

Questa e' una trasformazione **classica/statistica**, non una NN.

### 4.2 Min-max normalization dell'input
Dopo Anscombe:

- si calcolano `x_min` e `x_max`;
- si normalizza l'immagine in circa `[0, 1]`.

Output:

- `x_norm`;
- `x_min`, `x_max`.

### 4.3 Min-max normalization del target
Sul target `AS` il notebook non applica Anscombe nel flusso principale.

Fa solo:

- `y_norm`;
- `y_min`, `y_max`.

Questo e' importante per il report:

- **l'input passa per Anscombe + min-max**;
- **il target passa solo per min-max**.

## Fase 5: suddivisione spaziale train/test
Funzione coinvolta: `generate_patch_coords_in_row_range`.

Il notebook non usa uno split casuale globale dei pixel o delle patch. Usa invece uno **spatial split** verticale:

- righe `0 .. split_row - 1` -> training;
- righe `split_row .. H - 1` -> validation/test.

Nel notebook eseguito compare:

- `split_row = 95`

Questo significa che la parte bassa dell'immagine e' trattata come area non vista in training.

Per ogni area vengono generate coordinate `(r0, c0)` per patch 2D:

- grandezza patch: `patch_size`;
- passo: `patch_stride`;
- se l'ultimo passo non arriva al bordo, il bordo viene comunque coperto aggiungendo l'ultima coordinata utile.

Scopo:

- evitare leakage spaziale;
- verificare se la rete generalizza su righe mai viste.

Questa e' ancora una fase **classica**.

## Fase 6: estrazione patch e data augmentation
Classe coinvolta: `PatchDataset`.

Ogni coordinata produce:

- `x_patch`;
- `y_patch`;
- shape finale `[1, patch_size, patch_size]`.

Se `use_augmentation = True`, il notebook applica in coppia a input e target:

- rotazioni di `90`, `180`, `270` gradi;
- flip orizzontale;
- flip verticale.

Questo non e' un filtro di restauro ma una tecnica classica di training per aumentare robustezza.

## Fase 7: addestramento della U-Net principale
Classe coinvolta: `UNet2D`.

Qui entra in gioco la **prima vera rete neurale** del progetto.

### Architettura
La rete e' una `U-Net` 2D classica:

- encoder con blocchi `Conv + BatchNorm + ReLU`;
- `MaxPool` per comprimere;
- bottleneck;
- decoder con `ConvTranspose2d`;
- skip connections tra encoder e decoder;
- convoluzione finale `1x1`.

Il notebook usa tipicamente:

- `in_ch = 1`;
- `out_ch = 1`;
- `base = 32`.

### Cosa riceve la rete
Input della rete:

- patch di `x_norm`, cioe' immagine rumorosa preprocessata.

Target della rete:

- patch di `y_norm`, cioe' immagine `AS` normalizzata.

### Loss
Configurabile:

- `MSE`;
- oppure `L1`.

### Ottimizzatore
- `Adam`.

### Checkpoint
Il notebook salva il modello migliore secondo `val_loss` in:

- `outputs/checkpoints/`

con tag che includono input, patch, loss ed epoche.

### Significato funzionale
Questa U-Net svolge il ruolo di:

- denoiser;
- restauratore;
- mappatore da dominio rumoroso (`PC` o `BS`) a dominio anatomico (`AS`).

## Fase 8: inferenza full-image della U-Net
Dopo il training, la ricostruzione non viene fatta su tutta l'immagine in un colpo solo, ma di nuovo a patch.

Per ogni patch dell'immagine completa:

1. si estrae `x_patch`;
2. si fa `model(x_patch)`;
3. la patch predetta viene accumulata in una mappa `acc`;
4. si incrementa `cnt` nella stessa zona;
5. alla fine si calcola la media `acc / cnt`.

Quindi la ricostruzione finale nasce da:

- **inferenza locale a patch**;
- **fusione con overlap-average**.

Output principali:

- `recon_input = x_norm`;
- `recon_pred = ricostruzione U-Net`;
- `recon_gt = y_norm`.

Questa fase e' mista:

- inferenza con **NN**;
- fusione overlap con metodo **classico**.

## Fase 9: valutazione quantitativa della ricostruzione
Il notebook misura la qualita' solo nella zona di test non vista:

- `test_input = recon_input[split_row:, :]`
- `test_pred = recon_pred[split_row:, :]`
- `test_gt = recon_gt[split_row:, :]`

Metriche usate:

- `PSNR`;
- `SSIM`.

Nel notebook salvato compaiono valori di esempio come:

- input vs GT: `PSNR ~ 6.79 dB`, `SSIM ~ 0.303`;
- predizione vs GT: `PSNR ~ 15.75 dB`, `SSIM ~ 0.391`.

Questa fase e' completamente **classica**: nessuna rete, solo misurazione delle prestazioni.

## Fase 10: ablazione temporale sul numero di frame
Qui il progetto studia cosa succede se invece di usare tutti i `1000` frame si usano solo i primi `N`.

Per ciascun `N` (ad esempio `10, 50, 100, 200, 500, 1000`):

1. si ricaricano i frame grezzi di `PC`;
2. si fa `sum(axis=0)` sui primi `N` frame;
3. si riapplica `Anscombe`;
4. si riapplica `minmax_apply` usando **gli stessi `x_min` e `x_max` del training**;
5. si ricostruisce l'immagine con la stessa U-Net;
6. si calcolano `PSNR` e `SSIM`.

Questa parte e' molto importante dal punto di vista metodologico:

- la rete non cambia;
- cambia solo quanta evidenza temporale le viene data in ingresso.

Quindi l'ablazione separa:

- effetto del numero di frame;
- effetto della rete.

## Fase 11: generazione pseudo-label dei bordi
Funzione coinvolta: `compute_edge_pseudolabel_from_as_patch`.

Questa fase apre un secondo ramo del progetto: non piu' restauro intensita', ma **edge detection anatomica**.

Il notebook non parte da annotazioni manuali dei bordi. Le costruisce dal target `AS`.

Per ogni patch del target:

1. calcola differenze finite `dx` e `dy`;
2. calcola la magnitudine del gradiente;
3. stima una soglia tramite quantile/percentile;
4. binarizza.

Output:

- maschera di bordo binaria pseudo-ground-truth.

Questo e' un metodo **classico**, non una rete.

## Fase 12: EdgeNet CS - bordi a partire dalla ricostruzione PC
Qui compare una **seconda rete neurale**.

Pipeline temporale:

1. si prende una patch `xb` dal ramo `PC`;
2. la `U-Net` principale produce `unet_pred`;
3. dal target `yb` si genera la pseudo-label edge;
4. una `UNet2D` piu' piccola (`base=16`) riceve `unet_pred`;
5. produce logits edge;
6. viene ottimizzata con `BCEWithLogitsLoss`.

In altre parole:

- prima NN: ricostruisce l'intensita';
- poi metodo classico: genera i bordi target;
- seconda NN: impara a predire quei bordi.

Checkpoint:

- `edgenet_cs` in `outputs/checkpoints/`.

## Fase 13: ramo BS parallelo
Il notebook replica quasi lo stesso schema anche per `BS`.

Ordine:

1. `BS` -> Anscombe + min-max;
2. training di `unet_bs`;
3. generazione pseudo-label da `AS`;
4. training di `edgenet_bs`.

Quindi il progetto non ha una sola rete, ma potenzialmente:

- `unet_pc`;
- `edgenet_cs`;
- `unet_bs`;
- `edgenet_bs`.

## Fase 14: inferenza edge su immagine completa
Una volta addestrate le reti edge:

1. si ricostruisce prima l'intensita' completa con la U-Net corrispondente;
2. la ricostruzione viene di nuovo spezzata in patch;
3. ogni patch passa nella EdgeNet;
4. si applica `sigmoid` ai logits;
5. si accumulano le probabilita' in overlap;
6. si media con `acc / cnt`;
7. si soglia la probabilita' finale.

Output:

- `edge_prob`;
- `edge_bin`.

Anche qui la logica e' mista:

- predizione con **NN**;
- fusione e sogliatura con metodi **classici**.

## Fase 15: metriche sui bordi
Il notebook usa almeno due famiglie di metriche:

- `Dice` come misura di overlap;
- `Boundary F-measure` con tolleranza in pixel.

La `Boundary F-measure` viene implementata dilatando i bordi con max-pooling per consentire piccoli errori di allineamento.

Questa fase e' **classica**.

## Fase 16: EdgeBWNet e ramo binario
Il notebook contiene anche un ramo aggiuntivo, opzionale, chiamato `EdgeBWNet`.

Idea:

1. si parte da una versione bianco/nero derivata dalla ricostruzione U-Net;
2. una piccola CNN sequenziale cerca di predire i bordi;
3. loss: `BCEWithLogitsLoss`;
4. inferenza full-image con patch + overlap.

Pero' nel notebook il flag:

- `RUN_EDGEBW_TRAINING = False`

quindi questo ramo puo' restare inattivo se manca il checkpoint.

Per il report e' utile specificarlo:

- **fa parte dell'architettura del notebook**;
- **non e' necessariamente attivo nell'esecuzione di default**.

## Fase 17: post-processing classico dei bordi
Dopo i modelli neurali, il notebook prova anche tecniche completamente classiche.

### 17.1 Contorno da maschera B/N
Funzione: `bw_outline_from_mask`.

Procedura:

1. soglia la maschera;
2. esegue una sorta di erosione tramite `max_pool` sul complemento;
3. calcola il contorno come XOR fra maschera ed erosione;
4. prova diversi spessori.

### 17.2 Sweep di filtri edge classici
Funzione: `_detect_edges_classic`.

Per ogni immagine sorgente selezionata, il notebook prova combinazioni di:

- blur medio (`avg_pool`) con kernel diversi;
- gradiente `L1` o `L2`;
- soglia percentile;
- dilatazione;
- erosione.

Le sorgenti possono includere:

- `AS` normalizzata;
- edge GT;
- output U-Net PC/BS;
- raw PC/BS con diversi numeri di frame.

Questa e' la parte dove il progetto confronta esplicitamente:

- approcci **NN-based**;
- approcci **filter-based / morphological**.

## Fase 18: salvataggio artefatti
Gli artefatti principali prodotti dal progetto sono:

### Checkpoint modelli
In `outputs/checkpoints/`:

- `unet_pc`;
- `unet_bs`;
- `edgenet_cs`;
- `edgenet_bs`;
- opzionalmente `edgebw`.

### Immagini di debug e report
In `outputs/edgemetrics/`:

- predizioni edge;
- overlay;
- immagini full-frame;
- sweep di spessore;
- risultati dei confronti classici.

## Quando viene usata una rete neurale e quando no

## Operazioni senza NN
- lettura file `.asc`;
- rimozione metadati;
- reshape dei dati temporali;
- stacking `sum/mean`;
- selezione `pc` / `bs` / `pcbs`;
- trasformata di Anscombe;
- normalizzazione min-max;
- generazione coordinate patch;
- data augmentation geometrica;
- overlap-average;
- PSNR;
- SSIM;
- gradienti per pseudo-label edge;
- sogliatura percentile;
- Dice;
- Boundary F-measure;
- erosione/dilatazione;
- estrazione contorno;
- edge sweep classico.

## Operazioni con NN
- `UNet2D` principale per ricostruzione intensita' da `PC`, `BS` o `PCBS`;
- `UNet2D` ridotta come `EdgeNet CS`;
- `UNet2D` ridotta come `EdgeNet BS`;
- `EdgeBWNet` per ramo binario opzionale.

## Ordine esatto dei processi che ricevono l'immagine

## Pipeline principale di restauro
1. file `.asc` -> lettura righe
2. righe `PC`/`BS` -> reshape temporale
3. frame temporali -> stacking `sum/mean`
4. immagine stacked -> scelta input (`pc`, `bs`, `pcbs`)
5. input scelto -> Anscombe
6. input trasformato -> min-max
7. target `AS` -> min-max
8. immagini normalizzate -> split spaziale
9. aree spaziali -> patch extraction
10. patch train -> augmentation
11. patch input -> `UNet2D`
12. patch predette -> loss contro patch target
13. modello migliore -> checkpoint
14. immagine completa normalizzata -> inferenza patch-by-patch
15. patch predette -> overlap-average
16. ricostruzione finale -> metriche PSNR/SSIM sulla zona test

## Pipeline edge guidata da NN
1. target `AS` normalizzato -> pseudo-label edge tramite gradienti
2. input `PC` o `BS` normalizzato -> `UNet` di ricostruzione
3. ricostruzione U-Net -> `EdgeNet`
4. `EdgeNet` -> probabilita' di bordo
5. probabilita' -> soglia
6. maschera edge -> metriche Dice/BF

## Pipeline edge classica
1. immagine sorgente scelta
2. eventuale blur medio
3. gradienti `L1` o `L2`
4. soglia percentile
5. eventuale dilatazione
6. eventuale erosione
7. bordi finali -> metriche

## Ruolo di ciascun blocco architetturale

| Blocco | Tipo | Ruolo |
| --- | --- | --- |
| `PescettiDataLoader` | Classico | Legge i file e costruisce matrici numeriche |
| `_stack_input_rows` | Classico | Collassa i dati temporali in immagini 2D |
| `PreProcessor.anscombe` | Classico | Stabilizza il rumore Poisson-like |
| `PreProcessor.minmax_norm` | Classico | Porta i dati in scala compatibile col training |
| `PatchDataset` | Classico | Estrae patch input/target e applica augmentation |
| `UNet2D` principale | NN | Ricostruisce immagini anatomiche da input rumorosi |
| overlap-average | Classico | Riunisce le patch predette in una sola immagine |
| `compute_edge_pseudolabel_from_as_patch` | Classico | Costruisce etichette edge senza annotazioni manuali |
| `EdgeNet` | NN | Predice bordi a partire dalla ricostruzione U-Net |
| `EdgeBWNet` | NN | Variante leggera su ingresso binario/grayscale |
| morfologia e sweep classico | Classico | Confronta o rifinisce i bordi senza usare reti |

## Osservazioni importanti per il report

### 1. Il progetto e' notebook-centric
Non c'e' una libreria Python separata con moduli dedicati: il notebook contiene sia definizioni, sia training, sia inferenza, sia visualizzazione.

### 2. Il cuore del progetto e' ibrido
Non e' "solo deep learning". La pipeline combina:

- manipolazione classica del dato;
- preprocessing statistico;
- reti neurali per ricostruzione e bordi;
- post-processing morfologico;
- metriche classiche.

### 3. L'immagine non entra subito nella NN
Prima di arrivare alla rete, passa da:

- parsing testo;
- ricostruzione 2D da righe temporali;
- stacking;
- Anscombe;
- min-max;
- patching.

### 4. Il ramo edge e' secondario rispetto al restauro
La prima missione del progetto e' ricostruire l'intensita'. Il rilevamento dei bordi viene dopo, come task derivato.

### 5. Alcune parti sono opzionali o dipendono dai checkpoint
Se esiste gia' un checkpoint, il notebook lo riusa. Alcuni rami, come `EdgeBWNet`, possono non partire affatto se il flag resta disattivato o se il checkpoint manca.

### 6. C'e' una piccola ambiguita' documentale
Nel notebook compare `val_split` nella config, ma il flusso vero usa soprattutto uno split spaziale fissato tramite `split_row`. Nel report conviene dire che la validazione effettiva e' realizzata tramite separazione per righe.

## Conclusione
Dal punto di vista architetturale, `ZebraRestorer` e' una pipeline in piu' stadi in cui l'immagine attraversa prima una serie di trasformazioni classiche necessarie a rendere coerente il dato, poi una o piu' reti neurali per ricostruzione e analisi, e infine moduli classici di valutazione e raffinamento.

La distinzione piu' importante da riportare e' questa:

- **prima della U-Net**: costruzione del dato, stacking, Anscombe, normalizzazione, patching;
- **nella U-Net**: ricostruzione dell'immagine anatomica;
- **dopo la U-Net**: metriche, pseudo-label edge, EdgeNet, morfologia, confronti con filtri classici.
