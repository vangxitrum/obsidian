# anydesk-black-screen-two-causes

**Fact:** On the i3/X11 workstation (`tuan-System-Product-Name`, 10.0.0.67, GTX 1050, nvidia
580.173.02), AnyDesk 8.0.2 showed **"waiting for image"** on the remote viewer. Two independent
bugs stacked. Diagnosed 2026-09-09.

## Cause 1 (blocks capture startup): no primary monitor
Backend logged, looping every ~50ms:
`base.monitor_info - Could not find a primary monitor`

The xrandr `primary` flag sat on **DVI-D-0, which is disconnected**. AnyDesk enumerates only
connected outputs (HDMI-0 output=445, DP-0 output=470) and stalls if none is primary. The i3
`exec_always xrandr` line set only `--right-of`, never `--primary`.

Fix, persisted in `~/.config/i3/config` (~line 253):
```
exec_always xrandr --output $monitor_left --primary --output $monitor_right --right-of $monitor_left
```

## Cause 2 (the real blocker): PMTU black hole
Path MTU to the peer was **1480**; `enp5s0` was **1500**, and the path returned *no* ICMP
frag-needed, so TCP never discovered the limit. Full-size video segments vanished.

Tell-tale signature - small traffic fine, large traffic wedged:
- clipboard (32 B) and opus audio worked; video never arrived
- `ss -tn` showed `Send-Q 51840` stuck to the peer while `ping` was 3.4ms / 0% loss
- `desk_rt.auto_adjust - Adjusting for a direct connection (5.70 kb/s)`

Probe: `ping -M do -s <n> <peer>` -> 1452 OK (MTU 1480), 1460 FAIL. 20-byte shortfall = an
IP-in-IP/GRE tunnel on the ISP path.

Fix applied:
```
sudo ip link set enp5s0 mtu 1480
sudo nmcli connection modify netplan-enp5s0 802-3-ethernet.mtu 1480   # persist
echo 'net.ipv4.tcp_mtu_probing = 1' | sudo tee /etc/sysctl.d/99-mtu-probing.conf
```
Result: measured link went **5.70 kb/s -> 6016 kb/s**, opus 65k -> 260k. Image restored.

## Debugging playbook
- Desktop session is X11 on **`:1`** (`:0` is the GDM greeter as user `gdm`). From SSH:
  `export DISPLAY=:1 XAUTHORITY=/run/user/1000/gdm/Xauthority`
- `xrandr` from a bare SSH shell says `Can't open display` - expected, not the bug.
- Live repro: `tail -n0 -F /var/log/anydesk.trace ~/.anydesk/anydesk.trace`
- Prove X capture is healthy independently:
  `ffmpeg -f x11grab -video_size 1920x1080 -i :1+0,0 -frames:v 1 out.png`
  (a real desktop grab is ~500 KB; a black frame is ~2 KB). picom running is NOT a problem.
- **Audio/clipboard working while video is dead => suspect MTU, not capture.**
- Monitors: `DP-0` = DELL P2317H (left, +0+0), `HDMI-0` = LG FHD (right, +1920+0).
