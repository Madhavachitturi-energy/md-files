# Display Forwarding from a System using noVNC

This setup enables **display forwarding from a system (e.g., Jetson) using TigerVNC and noVNC**, allowing remote access through a **web browser** without installing a VNC client.

---

# Dependencies

Install the required packages:

```bash
sudo apt install tigervnc-standalone-server tigervnc-common novnc websockify
```

---

# Start VNC Sessions

Create two virtual desktops:

```bash
vncserver :10 -geometry 1920x1080 -depth 24
vncserver :11 -geometry 1280x720 -depth 24
```

---

# Start noVNC Web Access

Expose the VNC sessions through a browser using **websockify**.

```bash
websockify -D --web=/usr/share/novnc/ 6080 localhost:5910
websockify -D --web=/usr/share/novnc/ 6081 localhost:5911
```

Access from a browser:

```
http://<JETSON_IP>:6080/vnc.html
http://<JETSON_IP>:6081/vnc.html
```

---

# Stop Sessions

```bash
vncserver -kill :10
vncserver -kill :11
sudo killall websockify
```

---

# Launch Script

The following script launches **two TigerVNC servers and exposes them via noVNC**.

```bash
#!/usr/bin/env bash
# =============================================================================
# Dual TigerVNC + noVNC startup script for Jetson (headless friendly)
# =============================================================================
#
# What this script does:
#   - Starts two independent virtual VNC desktops:
#       Session :10 → VNC port 5910 → noVNC web port 6080 (1920×1080)
#       Session :11 → VNC port 5911 → noVNC web port 6081 (1280×720)
#   - Uses websockify to make them accessible from any web browser
#   - Designed to work reliably even when no physical monitor is connected
#
# Requirements:
#   sudo apt install tigervnc-standalone-server tigervnc-common novnc websockify
#
# First time usage:
#   Run `vncpasswd` once to set a VNC password (it will be shared by both sessions)
#
# Usage:
#   ./start-vnc-servers.sh       → start both sessions
#   ./start-vnc-servers.sh stop  → kill both VNC servers and websockify processes
#
# Access URLs (replace with your Jetson's IP):
#   http://192.168.1.50:6080/vnc.html  → Full HD session
#   http://192.168.1.50:6081/vnc.html  → Smaller HD session
#
# =============================================================================

set -euo pipefail

# Configuration
VNC_DISPLAY_1=":10"
VNC_PORT_1=$((5900 + ${VNC_DISPLAY_1#:}))
WEBSOCKIFY_PORT_1="6080"
GEOMETRY_1="1920x1080"
DEPTH="24"

VNC_DISPLAY_2=":11"
VNC_PORT_2=$((5900 + ${VNC_DISPLAY_2#:}))
WEBSOCKIFY_PORT_2="6081"
GEOMETRY_2="1280x720"

NOVNC_WEB_PATH="/usr/share/novnc"
JETSON_IP="192.168.1.50"

start_session() {
    local display="$1"
    local geometry="$2"
    local vnc_port=$((5900 + ${display#:}))
    local ws_port="$3"

    echo "Starting VNC server on display $display ($geometry) → port $vnc_port"

    vncserver "$display" \
        -geometry "$geometry" \
        -depth "$DEPTH" \
        -alwaysshared \
        -fg >/dev/null 2>&1 &

    sleep 1.5

    echo "Starting websockify bridge → http://$JETSON_IP:$ws_port/vnc.html"

    websockify -D \
        --web="$NOVNC_WEB_PATH" \
        "$ws_port" \
        "localhost:$vnc_port"
}

stop_all() {
    echo "Stopping all VNC servers and websockify processes..."

    vncserver -kill "$VNC_DISPLAY_1" 2>/dev/null || true
    vncserver -kill "$VNC_DISPLAY_2" 2>/dev/null || true

    sudo killall -q websockify 2>/dev/null || true

    echo "All sessions stopped."
    exit 0
}

if [[ "${1:-}" == "stop" ]]; then
    stop_all
fi

vncserver -kill "$VNC_DISPLAY_1" 2>/dev/null || true
vncserver -kill "$VNC_DISPLAY_2" 2>/dev/null || true
sudo killall -q websockify 2>/dev/null || true

sleep 1

echo "Starting dual noVNC + TigerVNC sessions..."

start_session "$VNC_DISPLAY_1" "$GEOMETRY_1" "$WEBSOCKIFY_PORT_1"
start_session "$VNC_DISPLAY_2" "$GEOMETRY_2" "$WEBSOCKIFY_PORT_2"

echo "Session 1 → http://$JETSON_IP:$WEBSOCKIFY_PORT_1/vnc.html"
echo "Session 2 → http://$JETSON_IP:$WEBSOCKIFY_PORT_2/vnc.html"

exit 0
```

---

# Run the Script

Make it executable:

```bash
chmod +x start_vnc.sh
```

Launch:

```bash
./start_vnc.sh
```

Stop sessions:

```bash
./start_vnc.sh stop
```

---

# Summary

| Session | VNC Port | Web Port | Resolution |
|--------|---------|---------|-----------|
| :10 | 5910 | 6080 | 1920×1080 |
| :11 | 5911 | 6081 | 1280×720 |

Access through browser:

```
http://JETSON_IP:6080/vnc.html
http://JETSON_IP:6081/vnc.html
```
