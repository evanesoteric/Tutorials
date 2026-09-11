# CMaNGOS TBC + Playerbots — Rootless Podman on Arch

A reproducible single-player Burning Crusade (2.4.3 / build **8606**) server with ike3 AI playerbots.
Two pieces run as rootless Podman quadlets: **MariaDB** and the **CMaNGOS core** (realmd + mangosd).
Source, configs, SQL, and extracted client data all live as plain files on the host; only the
*execution environment* is containerized. Client runs on the same box under Wine + DXVK.

> **Model:** one toolchain image (`cmangos-tbc:local`) is used three ways — to **compile**, to
> **extract** client data, and to **run** realmd/mangosd. MariaDB uses the official image.

---

## Layout

```
~/Games/wow-tbc/
├── Containerfile            # the toolchain/runtime image
├── src/
│   ├── mangos/              # cmangos/mangos-tbc  (+ bots under src/modules/Bots)
│   └── tbc-db/              # cmangos/tbc-db  (world DB + InstallFullDB.sh)
├── build/                  # out-of-source build tree (persisted for incremental rebuilds)
├── install/                # CMAKE_INSTALL_PREFIX  → bin/, etc/, ...
├── client-data/            # extracted dbc/ maps/ vmaps/ mmaps/ Cameras/
├── mariadb-data/           # MariaDB volume
└── (quadlets live in ~/.config/containers/systemd/)
```

```bash
mkdir -p ~/Games/wow-tbc/{src,build,install,client-data,mariadb-data}
cd ~/Games/wow-tbc
```

Host packages you'll want for the DB-import step (the script runs host-side and talks to the
container over the published port):

```bash
sudo pacman -S --needed podman mariadb-clients git
```

---

## Phase 1 — Clone

Clone on the **host** (the SQL files and `InstallFullDB.sh` need to be here for the DB phase; the
build container just bind-mounts this in).

```bash
cd ~/Games/wow-tbc/src
git clone https://github.com/cmangos/mangos-tbc.git mangos
git clone https://github.com/cmangos/playerbots.git mangos/src/modules/Bots
git clone https://github.com/cmangos/tbc-db.git tbc-db
```

The bots **must** sit at `mangos/src/modules/Bots` — that path is what `BUILD_PLAYERBOTS` and the
DB scripts expect.

---

## Phase 2 — Toolchain image

`~/Games/wow-tbc/Containerfile`:

```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential gcc g++ cmake make git ca-certificates \
      libboost-all-dev libssl-dev libbz2-dev zlib1g-dev \
      default-libmysqlclient-dev libmariadb-dev \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /work
```

```bash
cd ~/Games/wow-tbc
podman build -t cmangos-tbc:local .
```

This single image carries both the build toolchain and the runtime libs, so binaries compiled in it
run in it without library-version hunting. (If you later want a slim runtime, split it multi-stage —
not worth it for one user.)

---

## Phase 3 — Compile core + bots

```bash
cd ~/Games/wow-tbc
podman run --rm -it \
  -v "$PWD/src:/work/src:Z" \
  -v "$PWD/build:/work/build:Z" \
  -v "$PWD/install:/opt/cmangos:Z" \
  cmangos-tbc:local bash -c '
    cd /work/build &&
    cmake /work/src/mangos \
      -DCMAKE_INSTALL_PREFIX=/opt/cmangos \
      -DBUILD_PLAYERBOTS=ON \
      -DBUILD_EXTRACTORS=ON \
      -DPCH=ON &&
    make -j"$(nproc)" &&
    make install'
```

> **Why the prefix is `/opt/cmangos`, not `/work/install`:** CMaNGOS bakes `SYSCONFDIR` (where the
> module configs like `ahbot.conf`/`aiplayerbot.conf` are loaded from) into the binaries at configure
> time, derived from `CMAKE_INSTALL_PREFIX`. The runtime containers mount `install/` at
> `/opt/cmangos` (Phase 5/9), so the prefix **must** be `/opt/cmangos` or those configs silently fail
> to load. We install *to* `/opt/cmangos` inside the build container (mounting host `install/` there),
> so the output still lands on the host while the baked path matches runtime. Verify with
> `strings install/bin/mangosd | grep /opt/cmangos/etc`.

On the 5600X (6c/12t) expect roughly 15–30 min for a clean build. `build/` is persisted, so later
`git pull` + re-run only recompiles what changed (but **changing the prefix forces a full rebuild** —
wipe `build/` first if you ever do). After this you have `install/bin/{mangosd,realmd}`, the extractor
tools under `install/bin/tools/`, and `install/etc/*.conf.dist`.

> The game and login servers build by default — this tree names those options `BUILD_GAME_SERVER`
> and `BUILD_LOGIN_SERVER` (both default `ON`), so mangosd and realmd come out without extra flags.
> Note the active playerbots module dir in this tree is `src/modules/PlayerBots/` (not `Bots/`) — that's
> the one whose `ahbot.conf`/`aiplayerbot.conf` dists you copy in Phase 7.

---

## Phase 4 — Extract client data

You need a real **2.4.3 (8606)** TBC client (the one with `Wow.exe` + `Data/*.MPQ`). The extractors
read the client's MPQs and produce `dbc/ maps/ vmaps/ mmaps/ Cameras/` — and **those five dirs** are
what go into `client-data/`, *not* the client itself.

> **Have the client but not the extracted dirs?** Then you can't skip — the raw client (Wow.exe, MPQs)
> is the *input*; `client-data/` wants the *output*. Run the extraction below.
>
> **Already extracted on the Dell?** Skip this phase — rsync the five dirs straight into
> `~/Games/wow-tbc/client-data/` (`rsync -av ev@dell:/path/to/run/{dbc,maps,vmaps,mmaps,Cameras} ~/Games/wow-tbc/client-data/`).
> They're not architecture-specific, and this skips the multi-hour mmap step.

Run the extraction **inside the container** — the tools are linked against the Debian build libs, not
your host. After `make install` they live in `install/bin/tools/` (`ad`, `vmap_extractor`,
`vmap_assembler`, `MoveMapGen`, and `ExtractResources.sh`, which drives them in the correct
map → vmap → mmap order). The script must run *from the client root*, so copy the tools in:

```bash
cd ~/Games/wow-tbc
podman run --rm -it \
  -v "$PWD/install:/opt/cmangos:Z" \
  -v "$HOME/Games/wow-tbc/client:/client:Z" \
  -w /client \
  cmangos-tbc:local bash -c '
    cp -r /opt/cmangos/bin/tools/* /client/ &&
    bash ./ExtractResources.sh'
```

Choose **extract everything**; it asks how many CPUs for mmaps — give it the max offered (you have 12
threads). **mmaps are the long pole** (an hour or two for TBC — fewer maps than WotLK) but the bots
need them for pathfinding, so don't skip them. The dirs are written into the client folder; move just
the five into place and clean up the rest:

```bash
cd ~/Games/WoW_TBC/client
mv dbc maps vmaps mmaps Cameras ~/Games/wow-tbc/client-data/
rm -rf Buildings ad vmap_extractor vmap_assembler MoveMapGen MoveMapGen.sh ExtractResources.sh offmesh.txt *.log
```

---

## Phase 5 — Compose file + MariaDB

> **OpenRC note:** Podman quadlets are a *systemd generator* and do nothing without systemd, so this
> runbook uses **podman-compose**, which is init-system-agnostic. All three services are defined once
> here; you start the DB now and the rest in Phase 9.

```bash
sudo pacman -S --needed podman-compose
```

`~/Games/wow-tbc/compose.yaml`:

```yaml
services:
  mariadb:
    image: docker.io/library/mariadb:11
    container_name: mariadb
    environment:
      MARIADB_ROOT_PASSWORD: changeme
    volumes:
      - ./mariadb-data:/var/lib/mysql:Z
    ports:
      - "127.0.0.1:3306:3306"
    networks: [wow]
    restart: on-failure

  realmd:
    image: cmangos-tbc:local
    container_name: realmd
    depends_on: [mariadb]
    working_dir: /opt/cmangos/bin
    command: ["./realmd", "-c", "/opt/cmangos/etc/realmd.conf"]
    volumes:
      - ./install:/opt/cmangos:Z
    ports:
      - "3724:3724"
    networks: [wow]
    restart: on-failure

  mangosd:
    image: cmangos-tbc:local
    container_name: mangosd
    depends_on: [mariadb]
    stdin_open: true      # so `podman attach mangosd` gives the mangos> console
    tty: true
    working_dir: /opt/cmangos/bin
    command: ["./mangosd", "-c", "/opt/cmangos/etc/mangosd.conf"]
    volumes:
      - ./install:/opt/cmangos:Z
      - ./client-data:/opt/cmangos/data:Z
    ports:
      - "8085:8085"
    networks: [wow]
    restart: on-failure

networks:
  wow:
    name: wow      # force the literal name so `--network wow` (Phase 8) resolves
```

> `container_name` is pinned so the in-container DB host stays `mariadb`, and `networks.wow.name`
> stops podman-compose from prefixing the project name onto the network. `:Z` is a no-op on a
> non-SELinux Arch/Artix box — harmless to leave, fine to drop.

Bring up **just the DB** for the import, then create the app user:

```bash
cd ~/Games/wow-tbc
podman-compose up -d mariadb

# wait a few seconds for init, then:
mariadb -h 127.0.0.1 -P 3306 -u root -pchangeme <<'SQL'
CREATE USER IF NOT EXISTS 'mangos'@'%' IDENTIFIED BY 'mangos';
GRANT ALL PRIVILEGES ON *.* TO 'mangos'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
SQL
```

---

## Phase 6 — Import databases (incl. playerbots DB)

Run the world-DB installer **from the host**, pointed at the published port.

```bash
cd ~/Games/wow-tbc/src/tbc-db
./InstallFullDB.sh        # generates InstallFullDB.config, then exits the first time
```

Edit `InstallFullDB.config`. Set the connection + core path, name the three DBs, and **enable the
bots DB**:

```ini
CORE_PATH="/home/foobar/Games/wow-tbc/src/mangos"
DB_HOST="127.0.0.1"
DB_PORT="3306"
USERNAME="mangos"
PASSWORD="mangos"
# DB names (keep these consistent with your .conf files in Phase 7):
#   realmd / world / characters → tbcrealmd / tbcmangos / tbccharacters
PLAYERBOTS_DB="YES"
```

Then run it again and pick **`4) Full installation`** (fresh build). To (re)do *only* the bots DB
later, it's **`5) Advanced DB management` → `8) Create and fill playerbots db`**.

> Manual alternative if you'd rather not use the script for bots: apply `src/mangos/src/modules/Bots/sql/`
> — files under `characters/` to the characters DB and `world/` to the world DB, using **only the
> `tbc/` subfolders**, never vanilla/wotlk.

**Critical realmlist fix** — the client gets redirected from realmd to mangosd using whatever address
is in this row. Since everything is on this box:

```bash
mariadb -h 127.0.0.1 -u mangos -pmangos tbcrealmd \
  -e "UPDATE realmlist SET address='127.0.0.1', port=8085 WHERE id=1;"
```

---

## Phase 7 — Config files

Copy the dists in `install/etc/` to live configs and edit. CMaNGOS DB strings are
`host;port;user;pass;db`. **Use `mariadb` as the host** — that's the container name on the `wow`
network (the containers talk to each other by name, not via 127.0.0.1).

```bash
cd ~/Games/wow-tbc/install/etc
for f in mangosd realmd; do cp ${f}.conf.dist ${f}.conf; done
```

**realmd.conf**
```
LoginDatabaseInfo = "mariadb;3306;mangos;mangos;tbcrealmd"
```

**mangosd.conf**
```
LoginDatabaseInfo     = "mariadb;3306;mangos;mangos;tbcrealmd"
WorldDatabaseInfo     = "mariadb;3306;mangos;mangos;tbcmangos"
CharacterDatabaseInfo = "mariadb;3306;mangos;mangos;tbccharacters"
DataDir               = "/opt/cmangos/data"
```

**Playerbots config** — copy the TBC dist and make sure mangosd loads it. The dist is per-expansion:

```bash
cp ~/Games/wow-tbc/src/mangos/src/modules/Bots/playerbot/aiplayerbottbc.conf.dist \
   ~/Games/wow-tbc/install/etc/aiplayerbot.conf
```

> Confirm the `AiPlayerbot.ConfigFile` value in `mangosd.conf` matches the filename you used
> (`aiplayerbot.conf`). Then in `aiplayerbot.conf` enable bots and tune population:

```
AiPlayerbot.Enabled            = 1
AiPlayerbot.RandomBotAutologin = 1
AiPlayerbot.MinRandomBots      = 50
AiPlayerbot.MaxRandomBots      = 100      # start modest; raise once you see CPU headroom
```

Also set the playerbots DB connection inside `aiplayerbot.conf` to point at the bots database the
installer created (same `mariadb;3306;mangos;mangos;...` style). Start at 100 random bots — the 5600X
handles that easily; first launch *generates* their gear/characters so it's slow once, then cached.

---

## Phase 8 — First launch + account (interactive)

mangosd needs a console for `account` commands, so do the **first** start interactively on the `wow`
network (this also lets you watch bot generation):

```bash
podman run --rm -it \
  --name mangosd-init \
  --network wow \
  -v ~/Games/wow-tbc/install:/opt/cmangos:Z \
  -v ~/Games/wow-tbc/client-data:/opt/cmangos/data:Z \
  -w /opt/cmangos/bin \
  cmangos-tbc:local ./mangosd -c /opt/cmangos/etc/mangosd.conf
```

When you reach the `mangos>` prompt (press Enter if it's buried under Diff lines):

```
account create foobar YOURPASSWORD
account set gmlevel foobar 3 -1
account set addon foobar 1
```

`addon 1` = TBC. GM level 3 gives you all bot/world commands. `Ctrl-C` to stop once bots have
generated and you've made the account.

---

## Phase 9 — Run the server

Everything is already defined in `compose.yaml` (Phase 5). Bring the whole stack up:

```bash
cd ~/Games/wow-tbc
podman-compose up -d
podman logs -f mangosd          # watch startup; Ctrl-C just detaches the log
```

Need the `mangos>` console again later (more accounts, `.commands`)? `stdin_open`/`tty` are set, so:

```bash
podman attach mangosd           # Ctrl-P Ctrl-Q to detach without stopping it
```

**Play session** = `podman-compose up -d`; **done** = `podman-compose down`. Down = near-zero power,
which is the whole point versus the Dell.

### Optional: OpenRC service wrapper

Only if you want `rc-service cmangos start/stop` instead of running compose by hand. Rootless podman
from an init script needs the user's `XDG_RUNTIME_DIR`, which on Artix/elogind only exists while
you're logged in — so this is fine for on-demand control but **not** for true boot-time start unless
you switch this stack to **rootful** podman (run compose as root) to sidestep the runtime-dir issue.

`/etc/init.d/cmangos`:

```sh
#!/sbin/openrc-run
name="cmangos"
description="CMaNGOS TBC (rootless podman-compose)"

: "${cmangos_user:=ev}"
: "${cmangos_dir:=/home/foobar/Games/wow-tbc}"

depend() { need net; }

_run() {
    su - "$cmangos_user" -c \
      "export XDG_RUNTIME_DIR=/run/user/\$(id -u); cd '$cmangos_dir' && podman-compose $1"
}

start() { ebegin "Starting CMaNGOS"; _run "up -d";  eend $?; }
stop()  { ebegin "Stopping CMaNGOS"; _run "down";   eend $?; }
```

```bash
sudo chmod +x /etc/init.d/cmangos
sudo rc-service cmangos start
# sudo rc-update add cmangos default   # only if you really want it at boot (see caveat above)
```

---

## Phase 10 — Client (Wine + DXVK)

You already have the client decided; minimal checklist:

1. Point the client at the local server — edit `WTF/realmlist.wtf`:
   ```
   set realmlist 127.0.0.1
   ```
2. Run the 2.4.3 `WoW.exe` in a Wine prefix with **DXVK** installed (Lutris/Bottles, or
   `WINEPREFIX=... winetricks dxvk`). Plain Wine is fine for this DX9 client; DXVK → Vulkan on your
   6700XT (RADV) just makes it buttery and kills DX9 glitches.
3. Log in with the `ev` account you created.

Optional QoL addons (client-side): the playerbots **behavior addon**
(`celguar/mangosbot-addon`) and **EngBags inventory addon** (`davidonete/mangosbot-EngBags`) give you
in-game UI for ordering bots around instead of chat commands.

---

## Tuning / gotchas

- **Bots not spawning:** check `AiPlayerbot.Enabled=1`, `RandomBotAutologin=1`, the playerbots DB
  connection in `aiplayerbot.conf`, and that the playerbots DB actually populated (Phase 6).
- **Client connects to realmd but can't enter world:** almost always the `realmlist.address`/`port`
  row (Phase 6) — it must be `127.0.0.1` / `8085`.
- **Can't reach DB from a container:** containers use host `mariadb` (the container name), not
  `127.0.0.1`. Only the *host-side* installer/clients use `127.0.0.1:3306`.
- **CPU under load:** start at 100 random bots; each one is a simulated player. Watch
  `journalctl`/`htop` and raise `MaxRandomBots` if you've got headroom. Hundreds is doable but the 5600X
  is happiest in the low hundreds for a lively-but-smooth world.
- **Updating:** `git pull` in `src/mangos` (and the bots submodule) + `src/tbc-db`, re-run Phase 3,
  then apply any new DB updates via `InstallFullDB.sh` → option 2 (update) — back up `mariadb-data/` first.
```
