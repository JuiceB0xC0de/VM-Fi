·····························································
: █████   █████ ██████   ██████            ███████████ █████:
:░░███   ░░███ ░░██████ ██████            ░░███░░░░░░█░░███ :
: ░███    ░███  ░███░█████░███             ░███   █ ░  ░███ :
: ░███    ░███  ░███░░███ ░███  ██████████ ░███████    ░███ :
: ░░███   ███   ░███ ░░░  ░███ ░░░░░░░░░░  ░███░░░█    ░███ :
:  ░░░█████░    ░███      ░███             ░███  ░     ░███ :
:    ░░███      █████     █████            █████       █████:
:     ░░░      ░░░░░     ░░░░░            ░░░░░       ░░░░░ :
·····························································

## The Opening

Well, well, well. Look who finally solved a problem that's been sitting in the Reddit threads and pentesting forums like a bad smell since 2011.

"You can't do WiFi packet capture in a VM." Everyone knows this. Everyone's also been wrong the entire time.

A bartender with no real experience and an aversion to the word "no" decided enough was enough. As fun as it is to say "buy a dongle," those days are behind us.

Your Mac has WiFi. Your Kali VM has tools. Neither one talks to the other's radio stack directly. For over a decade, the internet's official answer was "buy a $70 dongle."

This bypasses that entirely.

---

## What This Actually Is

VM-Fi proves a simple principle: **elegance beats hardware limitations**.

You're going to:

1. Lock your Mac's WiFi to a channel and capture live 802.11 frames
2. Stream them through an ngrok TCP tunnel (encrypted, fast, works from anywhere)
3. Replay them into a virtual radio interface inside your Kali VM
4. Watch every pentesting tool you own suddenly believe it's sitting on real WiFi

No install scripts. No black boxes. Just bash, pipes, and honest data flow.

---

## Why This Matters

For years, people with VMs had two choices:
- Buy hardware (expensive, clunky, another thing to break)
- Use fake data (doesn't teach you anything real)

VM-Fi is option three: **real traffic, zero hardware, pure architecture**.

You already know ngrok and the principles of tunneling. You understand virtual interfaces. You know how 802.11 frames work. You connect the pieces together and suddenly WiFi analysis in a VM isn't a hardware problem anymore—it was an architecture problem. And it's solved.

That's the whole point. It isn't complicated.

---

## The System

```
macOS (has real WiFi)
    ↓
tcpdump locks en0 to channel 11
    ↓
captures live 802.11 radiotap frames
    ↓
pipes to netcat
    ↓
ngrok TCP tunnel (encrypted)
    ↓
Kali VM (listening)
    ↓
netcat writes to FIFO
    ↓
tcpreplay injects into virtual wlan0
    ↓
every Kali tool sees real monitor-mode traffic
```

---

## Prerequisites

**On macOS:**
- `tcpdump`
- `netcat` (built-in)
- ngrok CLI ([https://ngrok.com](https://ngrok.com))

**On Kali:**
- `netcat` (built-in)
- `tcpreplay` (`sudo apt install tcpreplay`)

**Network:**
- Both machines reach the ngrok tunnel

---

## Quick Start

### Kali: Set Up Virtual WiFi

```bash
sudo modprobe mac80211_hwsim radios=1
iw dev
```

You should see `wlan0` in managed mode.

### Kali: Flip to Monitor Mode

```bash
sudo nmcli device set wlan0 managed no
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
iw dev wlan0 info
```

Should show `type monitor`.

### Kali: Start the Listener Loop

**Terminal 1:**
```bash
mkfifo /tmp/radio.fifo
while true; do
  nc -lvkp 9000 >> /tmp/radio.fifo
done
```

**Terminal 2:**
```bash
sudo tcpreplay --intf1 wlan0 --enable-radiotap /tmp/radio.fifo
```

### Kali: Start ngrok Tunnel

```bash
ngrok tcp 9000
```

Copy the address it prints (e.g., `6.tcp.us-cal-1.ngrok.io:16039`).

### macOS: Capture and Send

```bash
sudo tcpdump --monitor-mode --select-channels=11 -i en0 -c 500 -w /tmp/sniff.cap
cat /tmp/sniff.cap | nc 6.tcp.us-cal-1.ngrok.io 16039
```

Replace the host/port with what ngrok printed. WiFi will reconnect automatically after tcpdump finishes.

### Kali: Watch It

**Terminal 3:**
```bash
sudo tshark -i wlan0
```

or

```bash
airodump-ng wlan0
```

You'll see live 802.11 frames from your Mac's capture replaying into the virtual interface.

---

## Troubleshooting

**"0 packets captured"**
- tcpdump locks you off the network while running
- Make sure you're on an active WiFi channel (1, 6, 11 for 2.4GHz)
- Use `-c 500` to force it to stop and reconnect

**"nc: missing hostname and port"**
- Don't use `tcp://` prefix
- Use: `nc 6.tcp.us-cal-1.ngrok.io 16039` ✓
- Not: `nc tcp://6.tcp.us-cal-1.ngrok.io:16039` ✗

**"Device or resource busy"**
```bash
sudo nmcli device set wlan0 managed no
```

**Nothing on the monitor interface**
- Make sure tcpreplay is running with `--enable-radiotap`
- Make sure the FIFO is being filled (check nc listener output)
- Make sure wlan0 is actually monitor mode (`iw dev wlan0 info`)

---

## Support the Work

If VM-Fi saved you a $70 dongle or helped you understand something new, consider buying me a coffee:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/V7V41HTV6T)

I went from pouring beer to learning to code in six months. Every cup helps keep the momentum going.

---

## License

MIT. Do whatever you want with it.

---

**Made by someone tired of being told "just buy a dongle."**

**Because understanding the problem beats buying a solution every time.**