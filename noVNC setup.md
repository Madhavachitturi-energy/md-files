“#!/usr/bin/env bash
# =============================================================================
# Dual TigerVNC + noVNC startup script for Jetson (headless friendly)
# =============================================================================
#
# What this script does:
#   - Starts two independent virtual VNC desktops:
#   	Session :10 → VNC port 5910 → noVNC web port 6080 (1920×1080)
#   	Session :11 → VNC port 5911 → noVNC web port 6081 (1280×720)
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
#   ./start-vnc-servers.sh      	→ start both sessions
#   ./start-vnc-servers.sh stop 	→ kill both VNC servers and websockify processes
#
# Access URLs (replace with your Jetson's IP):
#   http://192.168.1.50:6080/vnc.html 	→ Full HD session
#   http://192.168.1.50:6081/vnc.html 	→ Smaller HD session
#
# =============================================================================

set -euo pipefail

# ──────────────────────────────────────────────────────────────────────────────
#  Configuration
# ──────────────────────────────────────────────────────────────────────────────

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

# Your Jetson’s IP (used only for info messages – not required for functionality)
JETSON_IP="192.168.1.50"   # ← change this to your actual IP if you want nice messages

# ──────────────────────────────────────────────────────────────────────────────
#  Functions
# ──────────────────────────────────────────────────────────────────────────────

start_session() {
	local display="$1"
	local geometry="$2"
	local vnc_port=$((5900 + ${display#:}))
	local ws_port="$3"

	echo "Starting VNC server on display $display ($geometry) → port $vnc_port"

	# Start the virtual desktop (TigerVNC server)
	vncserver "$display" \
    	-geometry "$geometry" \
    	-depth "$DEPTH" \
    	-alwaysshared \
    	-fg >/dev/null 2>&1 &   # background – remove -fg if you want logs immediately

	sleep 1.5   # give vncserver a moment to initialize

	echo "Starting websockify bridge → http://$JETSON_IP:$ws_port/vnc.html"

	# Start websockify (daemon mode so script can exit cleanly)
	websockify -D \
    	--web="$NOVNC_WEB_PATH" \
    	"$ws_port" \
    	"localhost:$vnc_port"
}

stop_all() {
	echo "Stopping all VNC servers and websockify processes..."

	# Kill specific displays first (cleaner)
	vncserver -kill "$VNC_DISPLAY_1" 2>/dev/null || true
	vncserver -kill "$VNC_DISPLAY_2" 2>/dev/null || true

	# Kill any remaining websockify processes
	sudo killall -q websockify 2>/dev/null || true

	echo "All sessions stopped."
	exit 0
}

# ──────────────────────────────────────────────────────────────────────────────
#  Main logic
# ──────────────────────────────────────────────────────────────────────────────

if [[ "${1:-}" == "stop" ]]; then
	stop_all
fi

# Clean up any old/stuck sessions before starting fresh
vncserver -kill "$VNC_DISPLAY_1" 2>/dev/null || true
vncserver -kill "$VNC_DISPLAY_2" 2>/dev/null || true
sudo killall -q websockify 2>/dev/null || true

sleep 1

echo "Starting dual noVNC + TigerVNC sessions..."
echo "───────────────────────────────────────────────"

start_session "$VNC_DISPLAY_1" "$GEOMETRY_1" "$WEBSOCKIFY_PORT_1"
start_session "$VNC_DISPLAY_2" "$GEOMETRY_2" "$WEBSOCKIFY_PORT_2"

echo "───────────────────────────────────────────────"
echo "Session 1 (Full HD)  →  http://$JETSON_IP:$WEBSOCKIFY_PORT_1/vnc.html"
echo "Session 2 (Smaller)  →  http://$JETSON_IP:$WEBSOCKIFY_PORT_2/vnc.html"
echo ""
echo "To stop everything later, run:"
echo "	$0 stop"
echo "	# or manually:"
echo "	vncserver -kill $VNC_DISPLAY_1"
echo "	vncserver -kill $VNC_DISPLAY_2"
echo "	sudo killall websockify"
echo "───────────────────────────────────────────────"

exit 0”
