# REGOLE GLOBALI — MEMORIA CONDIVISA

**Progetto:** MusicAppMellon — lettore musicale locale per iOS, scritto in Flutter.
**Uso:** ogni agente legge questo file prima del proprio prompt. In caso di dubbio, prevale questo file.

## 1. Catena degli agenti

`0 Architetto → (1 Audio, 2 Dati, 3 Logica in parallelo) → 4 UI → Integrazione → 5 QA → bug loop`

## 2. Contratti

- `lib/domain/` è **congelato** dopo l'Architetto. Nessun altro lo modifica.
- `lib/domain/` non importa pacchetti esterni: solo Dart/Flutter base.
- Le implementazioni rispettano firme, nomi e tipi esatti delle interfacce.
- Se un contratto è insufficiente, l'agente lo segnala nel report: non lo modifica.

## 3. Iniezione e provider

- Le dipendenze (repository, servizi, `PathResolver`, adapter) sono iniettate per
  costruttore.
- Nessuna istanziazione di implementazioni di repository/servizi dentro un'altra classe.
  Gli helper puri senza I/O (es. `QueueController`) possono essere creati internamente.
- Esiste **una sola istanza** di `PathResolver`, condivisa da Audio, Dati e UI.
- I provider dei repository vivono solo in `lib/application/repository_providers.dart`
  con questi nomi fissi:
  - `playerRepositoryProvider`
  - `libraryRepositoryProvider`
  - `playlistRepositoryProvider`
  - `filePickerServiceProvider`
  - `pathResolverProvider`
- L'Architetto li crea restituendo i fake. La Logica li usa soltanto. Integrazione li
  sovrascrive nel composition root (`lib/composition/` + `main.dart`).
- Nessun altro agente li rinomina o crea provider duplicati.

## 4. Proprietà dei file

| Agente | Può scrivere in |
|---|---|
| 0 Architetto | `lib/domain/`, `lib/data/fake/`, `lib/application/repository_providers.dart`, `pubspec.yaml`, `analysis_options.yaml`, `CONTRACTS.md`, `reports/agente-0-architetto.md`, file generati da `flutter create` |
| 1 Audio | `lib/data/audio/`, `test/data/audio/`, `ios/Runner/Info.plist`, `MANUAL_TESTS_AUDIO.md`, `reports/agente-1-audio.md` |
| 2 Dati | `lib/data/library/`, `test/data/library/`, `test/fixtures/`, `MANUAL_TESTS_LIBRARY.md`, `reports/agente-2-dati.md` |
| 3 Logica | `lib/application/`, `test/application/`, `STATE_API.md`, `reports/agente-3-logica.md` |
| 4 UI | `lib/ui/`, `test/ui/`, `MANUAL_TESTS_UI.md`, `reports/agente-4-ui.md` |
| Integrazione | `lib/main.dart`, `lib/composition/`, `pubspec.yaml`, `pubspec.lock`, `ios/`, `INTEGRATION_REPORT.md`, `reports/integrazione.md` |
| 5 QA | `test/qa/`, `integration_test/`, `MANUAL_TESTS.md`, `BUGS.md`, `TEST_REPORT.md`, `reports/agente-5-qa.md` |

Regole trasversali:

- `main.dart` è di Integrazione. Nessun altro lo tocca.
- `pubspec.yaml` è dell'Architetto, poi solo Integrazione può aggiungere dipendenze.
  Gli altri le richiedono nel report.
- `CONTRACTS.md` lo crea l'Architetto; dopo di lui lo aggiorna solo il coordinatore.
  Gli altri agenti propongono modifiche nel report.
- Le istruzioni di cablaggio dei moduli vanno in `WIRING.md` nella cartella del modulo.
- Non lasciare codice di debug, override di test o trigger temporanei nel codice finale.

## 5. Modelli (contratto)

- `Song`: `id`, `title`, `artist`, `album`, `duration` (`Duration`), `relativePath`,
  `artworkPath?`. Immutabile, `==`/`hashCode`.
- `Playlist`: `id`, `name`, `songIds` (`List<String>`, ordinata, unmodifiable), `createdAt`.
- `PlaybackStatus`: `idle`, `loading`, `playing`, `paused`, `completed`, `error`.
- `PlayerState`: `status`, `currentSong?`, `position`, `duration`, `queue`
  (`List<Song>` unmodifiable), `currentIndex` (`int`). Immutabile, `==`/`hashCode`.
- `ImportResult`: `imported` (`List<Song>`), `skippedDuplicates` (`int`),
  `failed` (`List<String>`).

### Identificativi e percorsi

- `Song.id` = **SHA-256 lowercase hex del contenuto del file**. Il duplicato è un `id` già
  presente: non serve una colonna hash separata.
- `Playlist.id` = UUID v4 generato dai Dati.
- `relativePath` e `artworkPath` sono **relativi alla cartella Documents** dell'app.
  Mai percorsi assoluti salvati.
- `PathResolver` espone `documentsDirectory()` e `resolve(relativePath)`.

## 6. Semantica di riproduzione (congelata)

- Coda vuota: `currentSong == null`, `currentIndex == -1`, `position == 0`, `duration == 0`.
- `duration` è la durata effettiva del brano corrente (0 se sconosciuta); la UI la usa per
  lo slider, con fallback su `Song.duration`.
- La lista `queue` è la stessa istanza finché la coda non cambia: niente copie a ogni tick
  di posizione.
- `playSong(song, {queue})`:
  - con `queue` valorizzata: sostituisce la coda e cerca `song` per `id`. Se non c'è,
    lancia `ArgumentError`.
  - con `queue == null`: se `song.id` è già in coda mantiene la coda e salta al brano,
    altrimenti sostituisce la coda con `[song]`.
- `play()`:
  - da `idle`/`completed` con `currentSong == null`: parte dal primo brano della coda;
  - da `completed`: riparte dall'inizio del brano corrente;
  - da `paused`: riprende.
- `next()`: se esiste un brano successivo ci passa; altrimenti `status == completed`,
  `position == durata del brano corrente`, coda invariata.
- `previous()`: se `position > 3s` torna a zero; altrimenti va al brano precedente; sul
  primo brano torna a zero.
- `addToQueue(song)`: accoda. Non avvia la riproduzione, non cambia `currentIndex`.
- `insertNext(song)`: se c'è un brano corrente inserisce subito dopo `currentIndex`,
  altrimenti in testa. Non avvia la riproduzione e non cambia `currentIndex`.
- `removeFromQueue(index)`:
  - indice non valido → `RangeError`;
  - rimozione del brano corrente: passa al successivo se esiste, altrimenti `currentSong`
    null, `currentIndex -1`, `status idle`, `position 0`;
  - rimozione prima del corrente: `currentIndex--`;
  - rimozione dopo il corrente: `currentIndex` invariato.
- `reorderQueue(oldIndex, newIndex)`: gli indici si riferiscono alla coda **prima** del
  movimento; `newIndex` è la posizione finale **dopo** la rimozione. Se si sposta il brano
  corrente, `currentIndex` lo segue. La UI converte il `newIndex` di
  `ReorderableListView` (se `oldIndex < newIndex`, decrementa).
- Errore su un file: `status == error`, `currentSong` resta il brano fallito, `position 0`;
  se esiste un successivo si passa automaticamente, altrimenti resta `error`.
- `seek`: clamp tra zero e `duration`.
- `next()`, `previous()`, `seek()` a coda vuota: no-op.
- `dispose`: idempotente, chiude stream e player.

## 7. Semantica di import e libreria (congelata)

- Estensioni supportate (case-insensitive): `.mp3`, `.m4a`, `.aac`, `.wav`, `.flac`.
- Import **sequenziale**, un file alla volta.
- Per ogni file:
  1. se non esiste o non è supportato → `failed` con `"nomefile: motivo"`;
  2. SHA-256 in streaming; se l'`id` esiste già → `skippedDuplicates++`;
  3. copia in `Documents/audio/<id>.<ext>`;
  4. metadati con `audio_metadata_reader`; fallback:
     titolo = nome file senza estensione, artista = `"Artista sconosciuto"`,
     album = `"Album sconosciuto"`; durata = `Duration.zero` se non disponibile;
  5. se c'è una copertina, salvala in `Documents/artwork/<id>.<ext>` e registra
     `artworkPath`;
  6. inserimento in DB; se fallisce, elimina i file copiati (niente orfani).
- Un file che fallisce non blocca gli altri.
- `watchSongs()` ordinato per titolo (`COLLATE NOCASE`) e reattivo a ogni modifica.
- `deleteSong(id)`: elimina riga DB, file audio, copertina e riferimenti nelle playlist.
  Idempotente.

## 8. Semantica delle playlist (congelata)

- Nome con `trim`; vuoto o solo spazi → `ArgumentError`.
- Nomi duplicati consentiti.
- Un brano può comparire una sola volta per playlist: `addSong` su un brano già presente
  è un no-op.
- `position` è compatta `0..n-1` dopo ogni aggiunta/rimozione.
- `deletePlaylist` elimina a cascata le righe di `playlist_songs`.

## 9. Fake

- I fake in `lib/data/fake/` devono rispettare la semantica di questo file, non essere
  giocattoli: coda, `next`/`previous`, `insertNext`, rimozioni e riordino compresi.
- Devono contenere qualche brano e playlist di esempio per far girare la UI.
- Il fake di `FilePickerService` restituisce due percorsi di esempio; il fake di
  `LibraryRepository.importSongs` restituisce due brani di esempio e li aggiunge alla
  lista, aggiornando lo stream.

## 10. Report e documenti

Ogni agente scrive `reports/agente-<n>-<nome>.md` con questo formato:

```markdown
# Report — Agente N (Nome)
Stato: completato | parziale | bloccato

## File creati/modificati
## Decisioni
## Deviazioni dal prompt
## Richieste ai contratti
## Test eseguiti (comando + esito)
## Problemi aperti
## Cosa NON ho verificato
```

Altri documenti:

- `CONTRACTS.md`: modelli, interfacce, semantica, mappa di proprietà, provider.
- `STATE_API.md`: API pubblica della Logica per la UI.
- `WIRING.md` (Audio e Dati): come costruire l'implementazione e sostituire il fake.
- `INTEGRATION_REPORT.md`: esito dell'integrazione e verdetto "QA può partire: sì/no".
- `MANUAL_TESTS_*.md`: checklist dei moduli; QA le unisce in `MANUAL_TESTS.md`.
- `BUGS.md`, `TEST_REPORT.md`: output del QA.

## 11. Definition of Done (vale per tutti)

- `flutter analyze` senza warning.
- I test del proprio modulo sono verdi.
- Nessun codice di debug o override temporaneo.
- Report compilato in `reports/`.
- Se un requisito è ambiguo: scegli la soluzione più semplice e annotala nel report.

## 12. Bug loop

1. Il QA apre i bug in `BUGS.md` con modulo proprietario e gravità.
2. Il coordinatore fa il triage.
3. L'agente del modulo corregge e aggiorna il proprio report.
4. Integrazione rifà build + `analyze` + test + smoke.
5. QA round 2: tutti i bloccanti, le regressioni e rerun completo.
