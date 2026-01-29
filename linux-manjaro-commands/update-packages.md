# Manjaro Linux common commands etc.

## Fully update the system

``` bash
sudo pacman -Syu
```

This:
- Syncs package databases (-Sy)
- Upgrades all installed packages (-u)
- Keeps everything in sync (important on Manjaro)
- Never run pacman -Sy alone — that’s how partial upgrades break systems.

## Common Errors

1. **Package Database locked**: Pacman uses a lock file to prevent two package operations at once. Your update got interrupted → lock file stayed behind.

To fix follow the steps below:

``` bash

ps aux | grep pacman

sudo killall pacman

sudo rm /var/lib/pacman/db.lck

```

2. **Corrupted Keyring**: Usually happens when an update is interrupted (due to bad internet or power loss). 

To fix, follow the steps below:

``` bash

sudo pacman-key --init

sudo pacman-key --populate manjaro archlinux

# Update keyring package itself with newly populated keys
sudo pacman -Sy manjaro-keyring archlinux-keyring

# delete the corrupted/interrupted packages
sudo rm -rf /var/cache/pacman/pkg/*

# Re-run the update
sudo pacman -Syu --disable-download-timeout

```

### Why this happened (quick explanation)
- Update got interrupted
- Keyring wasn’t fully synced
- Pacman refuses to trust package signatures
- This is very common on Arch/Manjaro after network issues