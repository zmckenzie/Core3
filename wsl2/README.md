# Windows Subsystem for Linux 2

> **Recommended:** if you have Docker Desktop, use the one-command Docker setup instead:
> clone this repo inside WSL and run `./swgemu`. See
> [Quick Start: Docker](../README.md#quick-start-docker-windows--wsl2--docker-desktop-or-linux).
> The bootstrap below installs everything directly into a WSL distro and clones the upstream repository.

## Bootstrap

Open a powershell as administrator and type:

```
cd ~
curl -o bootstrap.bat https://swgemu.com/getcore3/?p=1
```

Then run the batch file:
```
.\bootstrap.bat
```

Follow the prompts. See the main [README.md](../README.md) for more details.
