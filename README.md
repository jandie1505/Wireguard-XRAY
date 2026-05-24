# Wireguard through XRAY / HTTPS
Tunnel wireguard through HTTPS using [XRAY/XTLS](https://github.com/XTLS/) to bypass VPN bans in restrictive networks.
  
Please note that this guide requires linux on both the server and the client, because it uses the linux routing tables and rules.

## Installation

### Server

#### 1. Have a working wireguard server

This guide does not explain how to setup a wireguard server. If you don't know how to setup a wireguard server, search the internet.

This guide assumes that you already have a working wireguard server.  
The wireguard server I have tested this with has IPv4 and IPv6 support and uses a `/etc/nftables/nftables.conf` firewall configuration. This might be irrelevant, but i write this here if it is not.

#### 2. Install Xray (on the server)

You need to install Xray.  
The recommended way is to use the [official install script](https://github.com/XTLS/Xray-install).

```shell
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl enable xray.service
sudo systemctl stop xray.service
```

#### 3. Configure Xray

You now need to edit the Xray configuration file.

It is most likely located at `/usr/local/etc/xray/xray.json` if you have used the official install script.

Use the [xray server config](xray/xray-server.json) as template and change the following placeholders:

| Placeholder | Description |
|--|--|
| `<CLIENT UUID>` | Each user has a UUID which works as an access "password" (so don't share it). You need to set one uuid per client. You can generate uuids using `uuidgen`. |
| `<PATH>` | The http path Xray is listening on. Example: `/api/v2/test` |


##### Changing wireguard server IP and ports

If your Xray server is not running on the same machine as the wireguard server or you are using different wireguard ports, you need to change them in the `"rules"` section (which is in the `"routing"` section):

In this case, there are two wireguard servers on the ports `51820` and `51821`.

```json
      {
        "type": "field",
        "ip": ["127.0.0.1/32"],
        "port": "<wireguard server 1 port>,<wireguard server 2 port>",
        "network": "udp",
        "outboundTag": "wireguard-only"
      }
```

##### Changing Xray HTTP port

You can also change the http server port in the `"inbounds"` section. In the given template, it is set to `10000`.

```json
      "port": 10000
```

#### 4. External access

Your Xray server must be accessible from the internet.

In order to use HTTPS, you need to setup a reverse proxy on port `443` which proxies the path you have specified in the `<PATH>` placeholder to the Xray http server. In the given config template, this is `127.0.0.1:10000`.

You only need to proxy the `<PATH>` to the Xray server. You can proxy the other paths to a static website or whatever you want.

I would recommend nginx or cloudflare tunnel.  
There is an [example nginx configuration](reverse-proxy/nginx.conf) in this repository.

You also need a valid SSL certificate to use HTTPS. You can use Let's encrypt for that.

#### 5. Finishing

Make sure your reverse proxy is accessible from the internet.  
When you try to access your server at `<YOUR DOMAIN>/<PATH>` (for example `your-domain.tld/api/v2/test`), you should see a blank page (no error).

Start Xray using:

```shell
sudo systemctl start xray.service
```

You don't need to modify your wireguard server configuration.

### Client

#### 1. Have a wireguard client config and a Xray server

This guide assumes that...

- ...you have an existing wireguard server and a working wireguard client configuration.
- ...you have set up an Xray server like in the [server section](#server).

#### 2. Install Xray (on the client)

You can install Xray like it is [installed on the server](#2-install-xray-on-the-server), but you probably should disable the service so that it does not run while you don't need it.

```shell
sudo systemctl disable --now xray.service
```

#### 3. Configure Xray

You now need to modify the Xray configuration file.

Like on the server, it is also most likely located at `/usr/local/etc/xray/xray.json` if you have used the official install script.

Use the [xray client config](xray/xray-client.json) as template:

| Placeholder | Description |
|--|--|
| `<SERVER ADDRESS>` | The address of your server. Reminder: There are two `<SERVER ADDRESS>` placeholders. You need to replace both with the same value. |
| `<CLIENT UUID>` | The UUID of your client which is also set in the server config. |
| `<PATH>` | The path your Xray server is accessible on. The value you have set in `<PATH>` on the server. |

You also need to set the `"port"` value in the `"inbounds"` section to the port of your wireguard server (the port which is inside your wireguard client configuration: `Endpoint=xxx`).

#### 4. Configuring wireguard

Copy your existing wireguard client configuration and change the following:

Add the `PreUp` and `PostDown` commands and set `Table` and `mtu`:

```conf
[Interface]
# [...]
Table = 1234
mtu = 1320

PreUp = ip rule add not from all fwmark 0x9 lookup 1234
PreUp = ip -6 rule add not from all fwmark 0x9 lookup 1234

PostDown = ip rule del not from all fwmark 0x9 lookup 1234
PostDown = ip -6 rule del not from all fwmark 0x9 lookup 1234
```

Change the `Endpoint` IP to `127.0.0.1` (your local Xray server):

```conf
[Peer]
# [...]
Endpoint = 127.0.0.1:<Port>
```

Here is an [example wireguard client configuration](wireguard/client.conf).

#### 5. Connecting and disconnecting

Connect:
```shell
sudo systemctl start xray.service
sudo wg-quick up <wireguard config>
```

Disconnect:
```shell
sudo wg-quick down <wireguard interface>
sudo systemctl stop xray.service
```

## How it works

### General

- The wireguard client on your computer connects to the local Xray server on your PC.  
- The local Xray server then connects to the remote Xray server behind your reverse proxy using HTTPS and tunnels the wireguard traffic from your computer to the remote Xray server.  
- Then, the remote Xray server sends the wireguard traffic to your actual wireguard server.

The firewall of your local network only sees normal HTTPS packets, which are most likely not blocked.

Please note that the extra layer of HTTPS tunneling will slow down performance, but in restrictive networks, it's better than no VPN connection.

### Networking on the client side

On the client side, there is the issue that the Xray HTTPS packets are tunneled through the wireguard connection, which is tunneled through Xray, which creates a loop and the server never receives the packets.

This is solved using ip rules from the linux kernel.

The local Xray server is configured to mark network packets with the fwmark `9`/`0x9` by setting `SO_MARK` for the outbound socket.

The wireguard client is configured to use the table 1234 (`Table=1234` in client config). This means that wireguard does not modify the default routes in the main routing table, but only those in the table with the ID 1234.  
This means that only traffic from the routing table with the ID 1234 is send through wireguard.

In the `PreUp` commands of the wireguard client config, there is now set up a rule which says "send all traffic through table 1234, except traffic which is marked with fwmark 0x9 (which is our Xray traffic)".

This allows Xray's HTTPS traffic to go out bypassing the tunnel, while everything else goes through the wireguard tunnel.
