# neko + LibreWolf

[neko](https://github.com/m1k1o/neko) (v3) with [LibreWolf](https://librewolf.net/) instead of Firefox. Upstream neko ships no
LibreWolf image; this app is built the same way as neko's own `apps/firefox`, on neko's base image.

- **Image:** `bluscream1/neko-librewolf` ([Docker Hub](https://hub.docker.com/r/bluscream1/neko-librewolf), amd64).
- **Preinstalled extensions:** uBlock Origin, Dark Reader, FoxMCP. Every other extension is allowed.
- **[FoxMCP](https://pypi.org/project/foxmcp/):** an MCP server, so AI agents can drive the browser (tabs, DOM, screenshots) at
  `http://<host>:8771/mcp`.
- **Persistent profile:** unlike LibreWolf's forget-everything default, cookies, logins and the session survive restarts
  (`librewolf.overrides.cfg`).
- **No letterboxing:** off, so the stream has no borders. WebGL is on.

## Quick start

| Method | Steps |
| --- | --- |
| Docker Compose | `apps/librewolf/compose.yaml`: set `NEKO_USER_PASSWORD` / `NEKO_ADMIN_PASSWORD`, `docker compose up -d`, open `http://<host>:8080`. |
| Unraid | Docker → Add Container → Template URL `https://raw.githubusercontent.com/Bluscream/neko-librewolf-docker/master/unraid/neko-librewolf.xml`, or copy [`unraid/neko-librewolf.xml`](../../unraid/neko-librewolf.xml) to `/boot/config/plugins/dockerMan/templates-user/my-neko-librewolf.xml`. |
| Build yourself | `docker build -t neko-librewolf apps/librewolf` (optional `--build-arg BASE_IMAGE=…`, `--build-arg FOXMCP_VERSION=…`). |

**Required:** `--shm-size=2g` (the browser crashes with the 64 MB default) and `--cap-add=SYS_ADMIN` (browser sandbox).

## Ports

| Container port | Protocol | Purpose |
| --- | --- | --- |
| 8080 | tcp | Web UI and API. Use TLS (reverse proxy, tunnel) outside the LAN. |
| 52100 | udp | WebRTC media. The host port **must equal** `NEKO_WEBRTC_UDPMUX`. |
| 52100 | tcp | WebRTC TCP fallback. The host port **must equal** `NEKO_WEBRTC_TCPMUX`. |
| 8771 | tcp | FoxMCP (MCP over HTTP). Optional. Whoever reaches it controls the browser, so bind it to a trusted address only. |

## Environment variables

These are the important ones. Every neko v3 setting can be set as `NEKO_<SECTION>_<KEY>`; see the
[neko configuration docs](https://neko.m1k1o.net/docs/v3/configuration).

| Variable | Default | Description |
| --- | --- | --- |
| `NEKO_MEMBER_PROVIDER` | `multiuser` | `multiuser` = two shared passwords. |
| `NEKO_MEMBER_MULTIUSER_USER_PASSWORD` | — | Viewer password. **Set it.** |
| `NEKO_MEMBER_MULTIUSER_ADMIN_PASSWORD` | — | Admin password (take control, kick, settings). **Set it.** |
| `NEKO_DESKTOP_SCREEN` | `1280x720@30` | Virtual screen `WxH@FPS`, e.g. `1920x1080@30`. |
| `NEKO_WEBRTC_UDPMUX` | — | Single UDP port for all WebRTC traffic (e.g. `52100`). |
| `NEKO_WEBRTC_TCPMUX` | — | Single TCP port for the WebRTC fallback (e.g. `52100`). |
| `NEKO_WEBRTC_EPR` | — | Alternative to the mux ports: a range such as `59000-59100` (publish the whole range). |
| `NEKO_WEBRTC_NAT1TO1` | auto | IP that viewers use to reach the WebRTC port (LAN IP for LAN use, public IP otherwise). If it is empty, it is detected through `NEKO_WEBRTC_IP_RETRIEVAL_URL` at start. A failed lookup gives a black picture, so set it explicitly when you know the IP. |
| `NEKO_WEBRTC_IP_RETRIEVAL_URL` | `https://checkip.amazonaws.com` | Used for the auto-detection. |
| `NEKO_WEBRTC_ICESERVERS_FRONTEND` | Google STUN | JSON list of STUN/TURN servers for viewers. Add a TURN server for viewers behind strict firewalls. |
| `NEKO_WEBRTC_ICESERVERS_BACKEND` | Google STUN | The same, for the server side. |
| `NEKO_CAPTURE_BROADCAST_URL` | — | RTMP target for broadcasting the desktop (to Restreamer, Owncast, Twitch, …). |
| `NEKO_CAPTURE_BROADCAST_AUTOSTART` | `true` | Start the broadcast with neko when a URL is set. Set `false` to start it from the admin menu or `POST /api/room/broadcast/start`. |
| `NEKO_CAPTURE_BROADCAST_PIPELINE` | built-in | Custom GStreamer pipeline, see below. |
| `NEKO_SERVER_BIND` | `:8080` (set by the base image) | Listen address of the web UI. |

### Broadcast pipeline for HLS / VRChat players (low latency)
neko's default broadcast is H.264 4:4:4, which many HLS players (VRChat, browsers) cannot decode. This pipeline sends
4:2:0, 25 fps, with a keyframe every second (good for 1-second HLS segments):

```
flvmux name=mux ! rtmpsink location='{url} live=1'
pulsesrc device={device} ! audio/x-raw,channels=2 ! audioconvert ! queue ! voaacenc bitrate=160000 ! mux.
ximagesrc display-name={display} show-pointer=true use-damage=false ! video/x-raw,framerate=25/1 ! videoconvert
! video/x-raw,format=I420 ! queue ! x264enc threads=4 bitrate=4500 key-int-max=25 vbv-buf-capacity=4500
byte-stream=true tune=zerolatency speed-preset=veryfast ! mux.
```
Put it on one line in Unraid. Adding `videorate` or x264 `option-string` made the pipeline stall silently in testing.

## Volumes

| Path in container | Purpose |
| --- | --- |
| `/home/neko/.librewolf/profile.default` | LibreWolf profile: logins, cookies, extensions. Owned by uid 1000 (`chown -R 1000:1000`). |
| `/home/neko/Downloads` | Downloads, uid 1000. |

## Files in this app

| File | Purpose |
| --- | --- |
| `Dockerfile` | LibreWolf from repo.librewolf.net, FoxMCP in a venv, socat. |
| `supervisord.conf` | Runs LibreWolf, openbox, FoxMCP and the 8771 forward. |
| `policies.json` | Enterprise policies: forced extensions, telemetry off, bookmarks. |
| `librewolf.overrides.cfg` | Persistent-session preferences (see above) plus neko's Firefox defaults. |
| `profiles.ini`, `openbox.xml` | Profile location, window manager (same as upstream Firefox). |

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Black picture, the UI loads | WebRTC cannot connect. Check `NEKO_WEBRTC_NAT1TO1`, that the port is forwarded (UDP + TCP), and that the host port equals the mux port. Viewers behind strict firewalls need TURN. |
| Browser tabs crash | `--shm-size` missing. |
| "Profile in use" | Remove `lock` / `.parentlock` from the profile folder while the container is stopped. |
| Broadcast "active" but nothing arrives | After a container rebuild, stop and start the broadcast again (admin menu or API). |
