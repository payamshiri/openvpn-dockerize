# openvpn-dockerize

A minimal, Docker-friendly way to run an OpenVPN server. This repository contains Dockerfile(s) and shell scripts to build and run an OpenVPN server in a container, with persistent configuration and keys.

Features
- Runs an OpenVPN server inside a container
- Persistent configuration and PKI via mounted volumes
- Simple docker run and docker-compose examples
- Intended for small deployments, testing, and learning

Prerequisites
- Docker (Engine)
- Docker Compose (optional)
- Capability to map the TUN device and provide NET_ADMIN capabilities to the container

Quick start (build + run)
1. Build the image:
   docker build -t openvpn-dockerize .

2. Create a directory on the host to persist configuration and keys:
   mkdir -p /srv/openvpn-data

3. Run the container:
   docker run -d \
     --name openvpn \
     --cap-add=NET_ADMIN \
     --device /dev/net/tun \
     -v /srv/openvpn-data:/etc/openvpn \
     -p 1194:1194/udp \
     -e OVPN_NETWORK=10.8.0.0/24 \
     -e OVPN_PORT=1194 \
     -e OVPN_PROTO=udp \
     openvpn-dockerize

Notes:
- --cap-add=NET_ADMIN and --device /dev/net/tun are commonly required for OpenVPN to create the tun device.
- Persisting /etc/openvpn (or the path used by the container) ensures keys and configurations survive container restarts.

Environment variables (recommended / common)
- OVPN_NETWORK — VPN subnet (example: 10.8.0.0/24)
- OVPN_PORT — UDP/TCP port to listen on (default 1194)
- OVPN_PROTO — Protocol (udp or tcp)
- OVPN_SERVER_NAME — server common name (optional)
- EASYRSA_BATCH — set to 1 to skip interactive prompts when initializing PKI
- OVPN_CLIENT_NAME — default client name to create
- OVPN_DNS — DNS server(s) pushed to clients (comma-separated)

Adjust these to match the actual variables implemented in this repo.

Docker Compose example
version: "3.8"
services:
  openvpn:
    image: openvpn-dockerize
    container_name: openvpn
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - "1194:1194/udp"
    volumes:
      - ./openvpn-data:/etc/openvpn
    environment:
      - OVPN_NETWORK=10.8.0.0/24
      - OVPN_PORT=1194
      - OVPN_PROTO=udp

Initializing PKI & generating client profiles
The exact commands depend on the helper scripts present in this repository. Typical tasks:
1. Initialize PKI / generate server certificates (run once):
   - Often implemented as an entrypoint action if /etc/openvpn is empty, or by running an init script inside the container.
   - Example (container-specific):
     docker exec -it openvpn /bin/sh -c "easyrsa init-pki && easyrsa build-ca nopass && easyrsa build-server-full server nopass"

2. Generate a client certificate/profile:
   - docker exec -it openvpn /bin/sh -c "easyrsa build-client-full CLIENTNAME nopass"
   - Collect keys and create a .ovpn client file (combine client cert, key, CA and remote/server info).
   - Some images include a helper like ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn

Check the repository scripts to see the exact command names and use those instead of the generic examples above.

Ports & networking
- Default: 1194/udp (changeable via OVPN_PORT and OVPN_PROTO)
- If running behind NAT, forward the chosen port from your router to the host running the container.
- Consider firewall rules and ensure UDP/TCP passthrough as configured.

Persistence & backups
- Back up the mounted data directory (e.g., /srv/openvpn-data). It contains PKI, server.conf, and client credentials.
- Losing the PKI means losing existing client certificates — keep backups.

Security notes
- Keep private keys safe; do not commit them to git.
- Consider using client passphrase protection for stronger security.
- Run images from trusted sources and keep base packages up to date.

Troubleshooting
- Permission errors with /dev/net/tun: ensure the device exists on the host and container has NET_ADMIN and device access.
- "TLS Error: TLS key negotiation failed": check server logs, ensure port/protocol and firewall/NAT are configured correctly.
- No routing to clients: verify server.conf pushes the correct routes and IP forwarding is enabled on the host.

Contributing
- Feel free to open issues and PRs.
- If you add features, update this README with new environment variables and usage examples.

License
This repository is provided under the MIT License. See the LICENSE file for details.
