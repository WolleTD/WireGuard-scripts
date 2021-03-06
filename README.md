# wg-setup WireGuard Management

I designed `wg-setup` to simplify the setup of WireGuard when handling large amounts of
clients. The server-side tool was initially built for use with systemd-networkd, but I
added wg-quick support to be able to test it in containers.

The `wg-setup-client` tool will use systemd-networkd as well to configure WireGuard, if
the service is enabled. Otherwise, it will fall back to wg-quick (but still enable a systemd
service for that, if systemd is available). Despite it's name, it's also capable of setting
up servers, as a WireGuard server is just a peer with a fixed listening port.

## Quick example

Set up a VPN connection with eight commands (of which three are signature checking)!

Server:
```
$ git clone https://github.com/WolleTD/wg-setup.git && cd wg-setup

# To install only wg-setup scripts and services
$ sudo ./install.sh

# To also configure a WireGuard server interface
$ sudo ./install.sh wireguard
```

Client:
```
$ gpg --recv-keys FB9DA662
$ curl -O https://raw.githubusercontent.com/WolleTD/wg-setup/$ref/wg-setup-client
$ curl -O https://raw.githubusercontent.com/WolleTD/wg-setup/$ref/wg-setup-client.sig
$ gpg --verify wg-setup-client.sig && chmod +x wg-setup-client
...
gpg: Good signature ...
$ sudo ./wg-setup-client -e vpn.example.com:12345 -p Vq12...3a4= 172.16.1.10/16
...
============================================================
WireGuard setup successful! Server side add-peer command:
wg-setup add-peer my_hostname my_publickey 172.16.1.10
```

Paste the last line printed by `wg-setup-client` in a terminal on the server and you're done.

## Manage peers with `wg-setup`

A WireGuard interface and thus a WireGuard server is usually configured in a file only editable
by root, also the running interface has to be restarted or configured separately to the file for
the change to take effect.
Doing this manually is tedious and error-prone, so `wg-setup` provides `add-peer` and `remove-peer`
commands to make this easier and safer.
`wg-setup` always updates both the interface and the backing configuration files and provides
an input validation layer for the file content.

While `wg-quick` supports an additional `SaveConfig` parameter, this only exports the running
interface configuration upon wg-quick shutdown, carrying the risk of losing new peer configuration
if the service or machine terminates abnormally.
Also, it simply doesn't work with `systemd-networkd`.

The `add-peer` command expects a hostname in addition to public key and IP address. This hostname
is stored in the configuration file as a comment, has to be unique and can be read by the
`list-peers` and `remove-peer` commands.
If the IP address is supplied without mask suffix, it will be defaulted to /32.

Both commands are interactive if no arguments are provided and confirmation can be skipped with
the `-y` option.

`list-peers` provides a convenient way to export the client list into some other format.
Currently, three formats are supported:

* `hosts`: an `/etc/hosts`-like list of names and IP addresses
* `pubkeys`: like `hosts` but with the pubkeys in a third column
* `dns`: BIND zonefile format (only host lines, no header)

### Hooks

`wg-setup` provides a simple hook mechanism, which will call all executables in
`WG_SETUP_HOOK_DIR` (which defaults to `/etc/wg-setup`) with these arguments:
`added|removed <name> <public key> <allowed ips>`. This can be used to e.g. update
a condensed list of hosts or a DNS zone. The example hooks don't even use the
provided arguments, but simply call `list-peers` to get a full list each time.

### add-peer example

```
$ wg-setup add-peer my-peer iCJSinvalidCG2/WP9D1/viGlv+WrNpWtd1XkRzyrFs= 172.16.0.10
[WireGuardPeer]
# my-peer
# Added by wolle at 2020-05-20
PublicKey = iCJSinvalidCG2/WP9D1/viGlv+WrNpWtd1XkRzyrFs=
AllowedIPs = 172.16.0.10/32

Add this configuration to /etc/systemd/network/90-wireguard.netdev? [Y/n]
```

### remove-peer example

```
$ wg-setup remove-peer 172.16.0.10
[WireGuardPeer]
# my-peer
# Added by wolle at 2020-05-20
PublicKey = iCJSinvalidCG2/WP9D1/viGlv+WrNpWtd1XkRzyrFs=
AllowedIPs = 172.16.0.10/32

Remove this configuration from /etc/systemd/network/90-wireguard.netdev? [y/N]
```

### list-peers

```
$ wg-setup list-peers pubkeys
172.16.0.10     my-peer         iCJSinvalidCG2/WP9D1/viGlv+WrNpWtd1XkRzyrFs=
...
```

## Setup clients with `wg-setup-client`

This is kind of a companion script. Where `wg-setup` shall help managing larger WireGuard
networks, a large amount of clients has to be setup as well.

Basic usage:
```bash
sudo ./wg-setup-client -e <server-endpoint> -p <server-pubkey> <ip-address>
```

- If `<ip-address>` is specified without prefix-length, `/24` is used
- If any of the three required parameters is missing, the values are queried interactively
- See `--help` for more options

##### Security

To easily setup "bare" clients, `wg-setup-client` can basically be executed from GitHub directly.
You may also serve your own copy, in any case you should _always_ incorporate some kind of
signature validation in such automatic setup environments and don't trust HTTP(S):

```bash
gpg --recv-keys FB9DA662
curl -O https://raw.githubusercontent.com/WolleTD/wg-setup/$ref/wg-setup-client
curl -O https://raw.githubusercontent.com/WolleTD/wg-setup/$ref/wg-setup-client.sig
gpg --verify wg-setup-client.sig && chmod +x setup-wg-quick
```
