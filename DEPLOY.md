# Deploy tego forka na serwerze `ai-assistant`

Ten plik dotyczy wyłącznie forka. Reszta repozytorium jest upstreamowa - nie zmieniamy jej.

## Co ten fork dokłada

Jeden plik zmieniony względem upstreamu, więc merge tagów nie powinien generować konfliktów:

- `docker-compose.yml`:
  - wolumen `~/.hermes:/opt/data` zmieniony na
    `${HERMES_DATA_DIR:-/data/hermes-agent}:/opt/data`, bo Coolify uruchamia
    `docker compose` w kontekście, w którym `~` nie wskazuje na katalog domowy
    użytkownika hosta.
  - dodatkowy wolumen `${SECOND_BRAIN_DIR:-/home/bartoszburaczewski/second-brain}:/opt/data/second-brain`
    - montuje osobne repo `bartekburaczewski/second-brain` (Obsidian vault, sklonowane
      do `~/second-brain` na hoście) do kontenera, żeby wbudowany skill `obsidian`
      mógł z niego bezpośrednio czytać/pisać. Wymaga ustawienia `OBSIDIAN_VAULT_PATH=/opt/data/second-brain`
      w `.env` kontenera - patrz sekcja "Pierwsza konfiguracja po wdrożeniu".
  - dodatkowy wolumen (read-only) `${SECOND_BRAIN_DEPLOY_KEY:-/home/bartoszburaczewski/.ssh/second_brain_deploy_key}:/opt/data/second-brain-deploy-key:ro`
    - dedykowany klucz deploy (read-write, tylko do repo `second-brain`, NIE
      osobisty klucz użytkownika) dla joba auto-sync - patrz "Auto-sync second-brain" niżej.
- ten plik

## Gałęzie

| Gałąź | Co to jest |
|---|---|
| `deploy` | **to, co jest wdrożone**: tag wydania upstreamu + nasza zmiana. Coolify śledzi tę gałąź. |
| `main` | gałąź `main` upstreamu, nietknięta. Nie deployujemy z niej. |

## Aktualizacja do nowego wydania

```bash
cd ~/dev/hermes-agent
git fetch upstream --tags
git tag -l 'v*' --sort=-v:refname | head -5     # co jest nowego
git checkout deploy
git merge v2026.X.X                              # tag, nie main
git push
```

Coolify przebuduje obraz i wdroży. Rollback: `git revert` merge'a albo przestawienie
gałęzi na poprzedni tag.

## Konfiguracja w Coolify

New Resource → Public Repository → `bartekburaczewski/hermes-agent`, gałąź `deploy`,
Build Pack: **Docker Compose**.

- Ports Exposes: brak (gateway łączy się wychodząco do Telegrama, nie potrzebuje portu)
- Domains: brak
- `network_mode: host` w compose - kontener współdzieli namespace sieciowy hosta,
  ale nic nowego nie jest przez to wystawione na zewnątrz (brak reguł w Cloudflare
  Tunnel/Access dla tego serwisu)

| Zmienna | Wymagana | Opis |
|---|---|---|
| `HERMES_UID` | zalecane | UID hosta, żeby pliki na wolumenie były czytelne poza kontenerem (`id -u`) |
| `HERMES_GID` | zalecane | j.w. dla GID (`id -g`) |
| `HERMES_DATA_DIR` | nie | override ścieżki na hoście dla `/opt/data`, domyślnie `/data/hermes-agent` |
| `SECOND_BRAIN_DIR` | nie | override ścieżki na hoście dla `/opt/data/second-brain`, domyślnie `/home/bartoszburaczewski/second-brain` |
| `SECOND_BRAIN_DEPLOY_KEY` | nie | override ścieżki na hoście do klucza deploy, domyślnie `/home/bartoszburaczewski/.ssh/second_brain_deploy_key` |

Uwaga: `container_name` w compose (`hermes`, `hermes-dashboard`) jest kosmetyczny - Coolify
i tak nadaje własne nazwy kontenerów (`gateway-<id>`, `dashboard-<id>`). Sprawdź faktyczną
nazwę przez `docker ps` zamiast zakładać `hermes`.

## Pierwsza konfiguracja po wdrożeniu

Model (subskrypcja Claude Pro/Max - **nie** `hermes auth add anthropic --type oauth`,
to trafia w pulę "extra usage" zamiast w limit subskrypcji, patrz pamięć
`project_hermes_agent_setup`). Zamiast tego `claude setup-token` na hoście, wynikowy
token do `.env` kontenera jako `ANTHROPIC_TOKEN`.

Telegram gateway - `TELEGRAM_BOT_TOKEN` + `TELEGRAM_ALLOWED_USERS` w `.env` (prościej
niż interaktywny `hermes gateway setup`).

Second-brain vault - w `.env` kontenera:

```
OBSIDIAN_VAULT_PATH=/opt/data/second-brain
```

**Ważne:** wyłącz `memory` i `session_search` (bug #65365, patrz pamięć
`project_hermes_agent_setup`):

```bash
docker exec <container> hermes tools disable memory session_search --platform cli
docker exec <container> hermes tools disable memory session_search --platform telegram
```

## Auto-sync second-brain

Two-way git sync so the vault stays consistent between the server (Hermes) and the
user's laptop (Obsidian + Web Clipper, writes to `2-Inbox/`). Script at
`/opt/data/scripts/second-brain-sync.sh` inside the container, scheduled via a
`--no-agent` cron job (no LLM call, just runs the script and delivers stdout):

```bash
hermes cron create "every 15m" --name second-brain-sync \
  --script second-brain-sync.sh --no-agent --deliver origin
```

The script pulls first (rebase+autostash, picks up laptop-side clips), then commits
and pushes anything Hermes wrote since the last sync. Uses `GIT_SSH_COMMAND` pointed
at the mounted deploy key for that one process only - never touches the shared
`.git/config` (which would break the host's own git access to the same repo, since
`/opt/data/second-brain` is a bind mount of the same working tree).
