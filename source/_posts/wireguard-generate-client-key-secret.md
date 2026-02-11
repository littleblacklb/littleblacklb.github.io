---
title: Generate wireguard client key pair
---

# Generate wireguard client key pair

> <https://www.wireguard.com/quickstart/>

## Key Generation

WireGuard requires base64-encoded public and private keys. These can be generated using the [`wg(8)`](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8) utility:

`$ umask 077 $ wg genkey > privatekey`

This will create `privatekey` on stdout containing a new private key.

You can then derive your public key from your private key:

`$ wg pubkey < privatekey > publickey`

This will read `privatekey` from stdin and write the corresponding public key to `publickey` on stdout.

Of course, you can do this all at once:

`$ wg genkey | tee privatekey | wg pubkey > publickey`

## Configure `wg0.conf`

```conf
[Interface]
# Server VPN address with /24 subnet (10.0.0.0-255) to accommodate multiple clients
Address = 10.0.0.1/24
# Allow forwarding from WireGuard interface; Enable NAT masquerading for outbound traffic
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth2 -j MASQUERADE
# Remove forwarding rule; Remove NAT masquerading rule
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth2 -j MASQUERADE
ListenPort = <port>
PrivateKey = <your-server-private-key>

[Peer]
PublicKey = <your-client-public-key>
AllowedIPs = 10.0.0.2/32  # Client IP address
```
