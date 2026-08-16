# Deploy tego forka na serwerze `ai-assistant`

Ten plik dotyczy wyłącznie forka. Reszta repozytorium jest upstreamowa - nie zmieniamy jej.

## Co ten fork dokłada

Jeden plik zmieniony względem upstreamu, więc merge tagów nie powinien generować konfliktów:

- `docker-compose.yml` - wolumen `~/.hermes:/opt/data` zmieniony na
  `${HERMES_DATA_DIR:-/data/hermes-agent}:/opt/data`, bo Coolify uruchamia
  `docker compose` w kontekście, w którym `~` nie wskazuje na katalog domowy
  użytkownika hosta.
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

## Pierwsza konfiguracja po wdrożeniu

Model (subskrypcja Claude Pro/Max, paste-the-code flow, bez tunelu SSH):

```bash
docker exec -it hermes hermes setup
```

Telegram gateway (token z @BotFather):

```bash
docker exec -it hermes hermes gateway setup
```
