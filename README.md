# Pescetti

🐟 Repository per esperimenti di segmentazione e sviluppo modelli (es. U-Net) nel progetto **Pescetti**.

## ✨ Obiettivo

Costruire una pipeline chiara e ripetibile per:
- preprocessing dei dati
- training dei modelli
- valutazione e confronto esperimenti

## 🗂️ Struttura consigliata

- `notebooks/` - analisi, prove e visualizzazioni
- `src/` - codice riutilizzabile (training, preprocessing, utilities)
- `configs/` - configurazioni esperimenti
- `outputs/` - risultati locali non versionati

## 🛡️ Regole di versionamento

Il progetto usa un `.gitignore` restrittivo per evitare push accidentali di file pesanti o sensibili:

- 🧠 **modelli/checkpoint** (`*.pt`, `*.pth`, `*.onnx`, `*.h5`, `*.pkl`, ecc.)
- 📊 **dataset/tabellari** (`*.csv`, `*.asc`, `*.parquet`, ecc.)
- 🖼️ **immagini/media** (`*.png`, `*.jpg`, `*.jpeg`, `*.tiff`, `*.webp`, ecc.)
- 🧹 **cache e temporanei** (cache Python, checkpoint notebook, log)

Se ti serve tracciare eccezioni specifiche, aggiungi regole `!pattern` nel `.gitignore`.

## 🚀 Avvio rapido

1. Crea e attiva un ambiente virtuale Python.
2. Installa le dipendenze del progetto.
3. Esegui notebook o script di training/valutazione.

## ✅ Prima del push

Controlla sempre cosa verra` tracciato:

```bash
git status
git ls-files
```
