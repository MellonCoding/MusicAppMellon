# MEMORIA CONDIVISA — MusicAppMellon

Stato aggiornato al 2026-11-05. Questo file è mantenuto dal coordinatore.
Gli agenti leggono `REGOLE.md` e il proprio prompt; qui trovano stato e decisioni.

## Stato attuale

**Fase:** pronta per l'Agente 0 (Architetto).
**Prossimo passo:** lanciare l'Architetto con il briefing e validare `CONTRACTS.md`.

| Agente | Stato | Report |
|---|---|---|
| 0 Architetto | da fare | — |
| 1 Audio | in attesa | — |
| 2 Dati | in attesa | — |
| 3 Logica | in attesa | — |
| 4 UI | in attesa | — |
| Integrazione | in attesa | — |
| 5 QA | in attesa | — |

## Decisioni congelate

| # | Decisione | Motivo |
|---|---|---|
| D1 | 7 ruoli: Architetto + 6 agenti (Audio, Dati, Logica, UI, Integrazione, QA) | Integrazione è diventata un agente reale |
| D2 | Contratti in `lib/domain/`, nessuna dipendenza esterna | isolamento e testabilità |
| D3 | `PlayerRepository.insertNext(Song)` | evita il trucco `addToQueue + reorderQueue` e la race in Logica |
| D4 | `Song.id` = SHA-256 del contenuto | dedup automatico, niente colonna hash separata |
| D5 | `Playlist.id` = UUID v4 | id leggibili e stabili |
| D6 | `relativePath` e `artworkPath` relativi a Documents | sopravvivono al cambio di sandbox iOS |
| D7 | DB `sqflite` + `sqflite_common_ffi` per i test | semplice, senza codegen |
| D8 | Metadati con `audio_metadata_reader` 1.8.1 | attivo (ott 2026), copre mp3/m4a/flac/wav e artwork |
| D9 | Audio con `just_audio` + `audio_service` 0.18.19 | stabile e mantenuto; `just_audio_background` è ancora beta |
| D10 | Provider repository fissi in `lib/application/repository_providers.dart` | un solo punto di override |
| D11 | `bootstrap()` restituisce il `ProviderContainer` | main usa `UncontrolledProviderScope` |
| D12 | Ogni agente scrive `reports/agente-<n>-<nome>.md` | memoria leggibile dai file, niente copia-incolla |
| D13 | Bug loop: QA → modulo responsabile → Integrazione → QA2 | responsabilità chiara |
| D14 | `CONTRACTS.md`: lo crea l'Architetto, poi lo aggiorna solo il coordinatore | evita scritture concorrenti |
| D15 | `PlayerState.duration` | la UI ha la durata effettiva anche senza metadati |

## Ordine di lancio

1. Architetto → verifica DoD, lettura `CONTRACTS.md`, aggiornamento di questa memoria.
2. Audio + Dati + Logica in parallelo.
3. UI appena esiste lo scheletro di `STATE_API.md`.
4. Integrazione con i report e i `WIRING.md`.
5. QA.
6. Bug loop.

## Problemi aperti

| # | Problema | Chi |
|---|---|---|
| A1 | Verificare che `audio_metadata_reader` legga la durata; se no, `Song.duration = 0` e la UI usa `PlayerState.duration` | Architetto / Dati |
| A2 | Verificare la compatibilità `just_audio` + `audio_service` nella stessa risoluzione | Architetto |
| A3 | Scegliere il deployment target iOS minimo che soddisfa tutti i pacchetti | Architetto |
| A4 | `file_picker` su iOS restituisce copie temporanee: pulizia dopo l'import | Dati |
| A5 | Persistenza nei test di integrazione: ricreare container/DB, non riavviare davvero l'app | QA |

## Indice dei report

| Agente | File |
|---|---|
| 0 Architetto | `reports/agente-0-architetto.md` |
| 1 Audio | `reports/agente-1-audio.md` |
| 2 Dati | `reports/agente-2-dati.md` |
| 3 Logica | `reports/agente-3-logica.md` |
| 4 UI | `reports/agente-4-ui.md` |
| Integrazione | `reports/integrazione.md` |
| 5 QA | `reports/agente-5-qa.md` |

## Note

- Le sessioni degli agenti non condividono contesto: il passaggio di memoria avviene
  tramite `REGOLE.md`, `CONTRACTS.md`, `STATE_API.md`, i `WIRING.md` e i report.
- Dopo ogni agente il coordinatore aggiorna questa memoria e prepara il briefing successivo.
