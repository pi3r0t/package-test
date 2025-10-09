Verify NAT-PMP works inside the wireguard container

We want natpmpc to talk to the VPN gateway, not your home router, so we run it inside the wireguard container.

Install once (quick & dirty):

```docker exec -it wireguard sh -lc 'apk add --no-cache natpmpc'```

Test:

```
docker exec wireguard natpmpc -a 0 0 udp 60
```

You should see output containing something like: Mapped public port 5xxxx. If you do, you’re golden. (If it fails, see the Troubleshooting at the end.) 
Proton VPN

Optional “clean” approach later: make a tiny derived image or a sidecar with natpmpc preinstalled so it survives container recreates.

2) Put the updater script on the host

Create a folder and the script:

```
sudo mkdir -p /opt/transmission-port-sync
sudo nano /opt/transmission-port-sync/update_transmission_port.py
```

Paste this script (tailored to your stack paths and container names):

```
#!/usr/bin/env python3
import json, re, subprocess, sys
from pathlib import Path
from time import sleep

WIREGUARD = "wireguard"           # container_name from your stack
TRANSMISSION = "transmission"     # container_name from your stack
SETTINGS_JSON = Path("/mnt/media/devmon/sda1/Sypi/appdata/transmission/settings.json")

NATPMP_TCP = ["docker","exec",WIREGUARD,"natpmpc","-a","0","0","tcp","60"]
NATPMP_UDP = ["docker","exec",WIREGUARD,"natpmpc","-a","0","0","udp","60"]
PORT_RE = re.compile(r"Mapped public port (\\d+)")

def sh(cmd):
    return subprocess.check_output(cmd, stderr=subprocess.STDOUT).decode(errors="ignore")

def get_forwarded_port():
    try:
        sh(NATPMP_TCP)                  # harmless; ensures mapping exists
        out = sh(NATPMP_UDP)            # parse from UDP output
    except subprocess.CalledProcessError as e:
        print("[port-sync] natpmpc failed:\\n"+e.output.decode(errors="ignore"), file=sys.stderr)
        return None
    m = PORT_RE.search(out)
    return m.group(1) if m else None

def get_configured_port():
    try:
        with open(SETTINGS_JSON) as f:
            return str(json.load(f).get("peer-port"))
    except Exception as e:
        print(f"[port-sync] read settings.json failed: {e}", file=sys.stderr)
        return None

def set_transmission_port_live(port):
    # Uses USER/PASS env already set in your container to auth RPC.
    cmd = [
        "docker","exec",TRANSMISSION,"/bin/sh","-lc",
        'transmission-remote -n "${USER}:${PASS}" --port ' + str(port)
    ]
    subprocess.check_call(cmd)

def persist_settings_json(port):
    try:
        with open(SETTINGS_JSON, "r+", encoding="utf-8") as f:
            data = json.load(f)
            data["peer-port"] = int(port)
            data["port-forwarding-enabled"] = True
            f.seek(0); json.dump(data, f, indent=2); f.truncate()
        return True
    except Exception as e:
        print(f"[port-sync] write settings.json failed: {e}", file=sys.stderr)
        return False

def main():
    while True:
        fwd = get_forwarded_port()
        if not fwd:
            sleep(15); continue

        curr = get_configured_port()
        if curr == fwd:
            sleep(45); continue

        try:
            set_transmission_port_live(fwd)
            persist_settings_json(fwd)
            print(f"[port-sync] Updated Transmission peer-port {curr} -> {fwd}")
        except subprocess.CalledProcessError as e:
            print(f"[port-sync] Failed to set Transmission port: {e}", file=sys.stderr)

        sleep(45)

if __name__ == "__main__":
    main()

```

Save and make it executable:

```
sudo chmod +x /opt/transmission-port-sync/update_transmission_port.py
```

Why this script works for your setup

Your transmission service uses network_mode: "service:wireguard", so inbound peers arrive through the VPN tunnel, not host port publishing. We only need to keep Transmission’s peer-port equal to the NAT-PMP forwarded port. No stack updates, no restarts.


3) Run it as a systemd service (on the host)

Create the unit:

```
sudo tee /etc/systemd/system/update-transmission-port.service >/dev/null <<'EOF'
[Unit]
Description=Sync Transmission peer port with ProtonVPN forwarded port
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/python3 /opt/transmission-port-sync/update_transmission_port.py
Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target
EOF

```

Enable & start:

```
sudo systemctl daemon-reload
sudo systemctl enable --now update-transmission-port.service
sudo journalctl -u update-transmission-port.service -f
```
You should see logs like Updated Transmission peer-port 51413 -> 53xxx.

5) Quick verification checklist

NAT-PMP working?

```
docker exec wireguard natpmpc -a 0 0 udp 60
```
Look for Mapped public port …. 
Proton VPN

Transmission actually listening on that port?
```
docker exec transmission sh -lc 'transmission-remote -n "$USER:$PASS" -pt'
```
It should print the new peer port.

Traefik/web UI: unchanged; you already publish 9091 on wireguard and route with labels, which is correct for your topology.

5) Optional hardening (nice to have)

Persist natpmpc: either

bake a tiny image FROM lscr.io/linuxserver/wireguard:latest with apk add --no-cache natpmpc, or

run a sidecar:

```
docker run -d --name wg-natpmp --network container:wireguard alpine:3.20 sh -c "sleep infinity"
docker exec wg-natpmp sh -lc 'apk add --no-cache natpmpc'
```
Then change the script commands from docker exec wireguard … to docker exec wg-natpmp ….

Credentials: the linuxserver/transmission image supports USER/PASS envs for the interface/RPC; that’s fine, but you can also use the TRANSMISSION_RPC_* envs or set rpc-username/password in settings.json if you prefer.


Troubleshooting

natpmpc says it can’t contact the gateway
Ensure your WG config had NAT-PMP enabled and that you’re connected to a P2P server. Try again from inside the wireguard container. If needed, specify the NAT-PMP gateway explicitly (natpmpc -g <gateway> -a 0 0 udp 60), using the gateway you see in ip route from inside the container. 
Proton VPN

Transmission doesn’t update
Make sure transmission-remote is available (it is in linuxserver/transmission) and that RPC auth matches your envs. You can also temporarily disable auth to test, then re-enable. 
LinuxServer

Do I need to publish the peer port on the host?
No. With network_mode: "service:wireguard", inbound traffic comes through the VPN tunnel. Just keep the peer-port in sync.
