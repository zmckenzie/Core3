# SWGEmu Core3: Recreating the Classic Star Wars Galaxies Experience

## What is SWGEmu?

Star Wars Galaxies (SWG) was an immensely popular massively multiplayer online role-playing game (MMORPG) introduced by Sony Online Entertainment in 2003. Despite its eventual shutdown in 2011, SWG left a lasting legacy in the hearts of thousands of players.

The SWGEmu project aims to recapture the magic of the game by focusing on a specific milestone: Pre-CU, or Pre-Combat Upgrade. The Combat Upgrade introduced significant changes to the gameplay, which were met with disapproval by many devoted players. It was in response to these changes that the SWGEmu project was born—a collective effort to "recreate" the game as it existed during the Pre-CU era.

At the core of the SWGEmu project lies the "Emulator," referring to the meticulously crafted server software being developed by the SWGEmu team. This Emulator endeavors to faithfully mimic Sony Online Entertainment's server-side software, which once hosted the immersive galaxies of Star Wars Galaxies during the Pre-CU era. To achieve this ambitious goal, the SWGEmu team has dedicated countless hours to reverse-engineering the code from scratch. They have built their own custom engine and core game logic, ensuring that every detail is crafted with love and precision.

The SWGEmu project is not only a nostalgic journey for passionate players but also a testament to the dedication and perseverance of a talented team. By resurrecting the Pre-CU era, SWGEmu offers an opportunity for both veterans and newcomers to experience the rich and captivating universe of Star Wars Galaxies, as it was once cherished by thousands of fans.

If you have any questions, need support, or want to contribute to the SWGEmu project, please refer to the documentation and resources provided in this repository.

## Quick Start: Docker (Windows + WSL2 + Docker Desktop, or Linux)

One command builds the server from this checkout and runs it with its database:

```
./swgemu
```

It checks your setup, downloads the client `.tre` files, compiles Core3 in a container, starts
MariaDB and the server, waits until the galaxy is up, and prints how to connect.

### One-time setup on Windows

1. **Install WSL2 with Ubuntu.** In PowerShell (as Administrator): `wsl --install -d Ubuntu`, then reboot
   and create your Linux user when Ubuntu opens.
2. **Install [Docker Desktop](https://www.docker.com/products/docker-desktop/).** In *Settings → Resources →
   WSL integration*, turn on your Ubuntu distro, then *Apply & restart*.
3. **Give WSL enough memory** (recommended 8GB+). Create `%UserProfile%\.wslconfig` containing:
   ```
   [wsl2]
   memory=8GB
   ```
   then run `wsl --shutdown` in PowerShell and restart Docker Desktop.
4. **Clone inside WSL, not under `C:\`.** Open Ubuntu and run:
   ```
   sudo apt-get update && sudo apt-get install -y git
   cd ~
   git clone --recurse-submodules https://github.com/zmckenzie/Core3.git
   cd Core3
   ./swgemu
   ```
   Building from `/mnt/c/...` works, but it is several times slower.

The first build takes 20–60 minutes depending on your CPU. Later builds reuse a compile cache and
only rebuild what changed. The first server start also builds navmeshes, which takes a while.

### TRE files

The server needs the 51 `.tre` files from a Star Wars Galaxies client. `./swgemu` puts them in `./tre`.
If any are missing, the first run asks where to get them and saves the answer as `TRE_SOURCE` in `.env`.
That file is git-ignored, so links are never committed. Ask whoever runs the server for the link to the TRE
archive (a Google Drive file shared as *Anyone with the link*, about 3GB), or use files you already have:

```
./swgemu tre 'https://drive.google.com/file/d/<id>/view'   # a shared Google Drive file
./swgemu tre 'C:\Users\you\Downloads\tre.tar'     # a .tar/.tar.gz/.zip (Windows paths work)
./swgemu tre /mnt/c/SWGEmu                         # an installed client folder
```

If Google Drive refuses the download ("quota exceeded"), download the TAR in your browser and use the
first form above.

### Connecting

Edit `login.cfg` (or `swgemu_login.cfg`) in the client folder:

```
[ClientGame]
loginServerAddress0=127.0.0.1
loginServerPort0=44453
```

Accounts are created on first login. To make your account an admin, log in once, then run
`./swgemu admin <account>`.

**Playing with friends on your LAN or the internet:** set `GALAXY_ADDRESS` in `.env` to this PC's LAN IP
(from `ipconfig`) or your public IP, then run `./swgemu up`. Allow UDP 44453, 44462 and 44463 through Windows
Firewall (and forward them on your router for internet play). Friends use that address in their `login.cfg`.

### Everyday commands

| Command | What it does |
| --- | --- |
| `./swgemu` / `./swgemu up` | Build if needed, start, wait until ready |
| `./swgemu stop` | Clean shutdown (saves the world first) |
| `./swgemu logs` | Follow the server log |
| `./swgemu console` | Attach to the server console (Ctrl-C detaches, the server keeps running) |
| `./swgemu cmd save` | Send a single console command |
| `./swgemu admin <name>` | Give an account admin level 15 |
| `./swgemu status` | Containers, galaxy row, account count |
| `./swgemu update` | `git pull`, then rebuild and restart |
| `./swgemu backup` | Database dump + world objects into `./backups/` |
| `./swgemu doctor` | Diagnose Docker, memory, disk, line endings, TRE files |
| `./swgemu reset` | Delete all accounts, characters and world state |

Settings (galaxy name and address, admins, build type, log level) are in `.env`. Server config overrides go
in `server-config/*.lua` (see [server-config/README.md](server-config/README.md)).

### How it is put together

* `docker-compose.yml` runs two services: `db` (MariaDB 10.11, which loads `MMOCoreORB/sql/swgemu.sql`
  on first start) and `core3`.
* `docker/server/Dockerfile` compiles engine3 and Core3 from **this checkout** with clang 19 on Ubuntu 24.04,
  using ccache in a BuildKit cache mount, then copies the result into a slim runtime image.
* `docker/server/entrypoint.sh` checks the TRE files, waits for the database, points the galaxy row at
  `GALAXY_ADDRESS`, writes `conf/config-local.lua`, and runs core3 (under gdb by default, so crashes print a
  backtrace). On `docker stop` it sends `exit` to the console so the world is saved before shutdown.
* World data (`bin/databases`), navmeshes, logs and the database live in named Docker volumes, so they survive
  rebuilds. `./swgemu reset` deletes them.

### Troubleshooting

* **`docker: command not found` or "Cannot reach Docker" in WSL:** `./swgemu` prints the likely cause. The usual fixes:
  * Docker Desktop must be running and show *Engine running*.
  * In Docker Desktop, *Settings → General → Use the WSL 2 based engine* must be on, and
    *Settings → Resources → WSL integration* must be on for your distro. Click *Apply & restart*, then open a new WSL terminal.
  * The distro must be WSL 2: in PowerShell, `wsl -l -v` must show `VERSION 2` (if not, run `wsl --set-version Ubuntu 2`).
  * Don't install Docker inside the distro (`apt install docker.io` / `docker-ce`). It hides Docker Desktop's. Remove it with
    `sudo apt-get remove docker.io docker-ce docker-ce-cli`, then run `wsl --shutdown` in PowerShell.
  * "permission denied" on the socket: `sudo usermod -aG docker $USER`, then `wsl --shutdown`.
  * `DOCKER_HOST` must not be set, and `docker context show` should print `default`.
* **`bad interpreter` or `$'\r': command not found`:** the repo was cloned with Windows line endings. Clone
  it from inside WSL, or run `git config --global core.autocrlf input && git rm -rq --cached . && git reset -q --hard`.
* **Compiler killed / `signal 9` during build:** not enough memory. Raise `memory=` in `.wslconfig`, or set
  `BUILD_JOBS=2` in `.env`.
* **Client hangs at "connecting to server":** `GALAXY_ADDRESS` is not reachable from the client machine.

## Docker Development Container (legacy)

The original interactive development container is still in `docker/` (`docker/build.sh`, `docker/run.sh`).
It clones the upstream `swgemu/Core3` repository into a volume rather than using this checkout, and expects
the TRE files in a `shared-tre` Docker volume:

```
$ cd ~/SWGEmu
$ tar cf - *.tre | docker run -i --rm -v shared-tre:/tre debian:bookworm bash -c 'tar xvf - -C /tre'
$ cd docker && ./build.sh && ./run.sh
```

Inside the container, `build` compiles the server and `run` starts it under gdb in screen.

## Windows Subsystem for Linux Setup

There is an automated setup available for Windows 10+ with WSL.

See [wsl2/README.md](wsl2/README.md) for instructions.

## Linux Automated Setup

You can use the automated linux setup if you plan to run the server in a vm or dedicated cloud or metal server.

See [linux/README.md](linux/README.md) for instructions.

## Linux Manual Build

### Dependencies
  * Debian 12
  * CMake 3.18.0+
  * BerkeleyDB 5.3
  * MariaDb Client and Server
  * OpenSSL libraries
  * pthreads
  * Lua 5.3 libraries
  * Zlib libraries
  * clang19
  * java runtime
  * boost
  * ninja-build

#### Clang-19

The easiest way to get the clang19 to build with is to use the script provided by the llvm repo:

```
sudo -i
apt-get install -y apt-transport-https ca-certificates git gnupg lsb-release moreutils software-properties-common wget
wget -O /tmp/llvm.sh https://apt.llvm.org/llvm.sh
chmod +x /tmp/llvm.sh
/tmp/llvm.sh 19 all
(set +x;cd /usr/bin;for i in ../lib/llvm-*/bin/*; do ln -sfv $i .; done)
clang --version
ld.lld --version
exit
```

This will install the latest and symlink all the files to /usr/bin so CMake finds them etc.

### Build

  * Install dependencies (Debian 12)

        sudo apt install build-essential libmariadb-dev libmariadb-dev-compat liblua5.3-dev libdb5.3-dev libssl-dev cmake git default-jre libboost-all-dev gdb ninja-build libjemalloc-dev

  * Clone core3 repository somewhere  (~/workspace)

        mkdir -p ~/workspace
        cd ~/workspace
        git clone https://review.swgemu.com/Core3

  * Build Core3 For Development

        cd Core3/MMOCoreORB
        make build-ninja-debug

  * Build Core3 For Production

        cd Core3/MMOCoreORB
        make -j$(nproc)

  * Import sql database into mariadb

        sudo apt-get install mariadb-server
        Setup MariaDB User, database and import sql/swgemu.sql

### How to Run

    * Navigate to the Directory

        cd ~/workspace/Core3/MMOCoreORB/bin

    * Load GDB

        gdb ./core3

    * Start Core3

        r

# License

    Copyright (C) 2019 SWGEmu

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU Affero General Public License as published by the Free Software Foundation,
    either version 3 of the License, or (at your option) any later version.

    This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
    without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
    See the GNU Affero General Public License for more details.

    You should have received a copy of the GNU Affero General Public License along with this program.
    If not, see <http://www.gnu.org/licenses/>.

For more information, see [https://review.swgemu.com/](https://www.docker.com/products/docker-desktop/) or our official [Development Discussion](https://www.swgemu.com/forums/forumdisplay.php?f=10) forums.

# Donate

The SWGEmu project is a 100% volunteer effort and funded by community donations. Consider [donating](https://www.swgemu.com/donate/?rm) to help keep the project moving forward. We use the donations to pay for our infrastructure and services to support the development process and running test servers.
