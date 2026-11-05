## Cosa è l'app

Un **lettore musicale per iOS scritto in Flutter**, pensato come banco di prova per una futura app. Riproduce i brani che l'utente importa dall'app File di iOS, tutto in locale: niente streaming, niente account.

## Funzionalità principali

- **Import dei brani:** selezione multipla di file (mp3, m4a/aac, wav, flac) dall'app File. I file vengono copiati nella cartella dell'app, i metadati (titolo, artista, album, copertina) letti dai tag, i duplicati scartati.
- **Libreria:** elenco di tutti i brani, con possibilità di riprodurli, aggiungerli alla coda o a una playlist, ed eliminarli.
- **Riproduzione:** play/pausa, avanti, indietro, slider di posizione. Continua con l'app in background e si controlla da lock screen e Control Center.
- **Coda:** aggiungi un brano, "Riproduci dopo", riordina trascinando, rimuovi con swipe.
- **Playlist:** crea, rinomina, elimina, aggiungi e rimuovi brani, riproduci l'intera playlist o accodala.
- **Interfaccia:** tab Libreria e Playlist, mini-player sempre visibile, player a schermo intero, tema chiaro e scuro, tutto in italiano.

## Come è costruita

| Strato | Compito |
|---|---|
| `domain/` | Modelli e interfacce (i contratti) |
| `data/` | Motore audio, database, import dei file |
| `application/` | Stato e logica (Riverpod) |
| `ui/` | Schermate e widget |

## Catena degli agenti

**0 Architetto** → **1 Audio, 2 Dati, 3 Stato** (in parallelo) → **4 UI** → **Integrazione** → **5 QA** → correzione bug → nuovo giro di QA.

Fuori scope per ora: streaming, cloud, Android, equalizer, testi dei brani.