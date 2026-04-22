# Technical Deep Dive

This document explains exactly how the vulnerability works, why common mitigations fail, and what a correct fix looks like.

---

## How xray and sing-box set up local proxying

When you start a VLESS client on Android the app needs a way to intercept all your device's traffic and route it through the tunnel. Android does not let apps do this directly, so the client uses the Android VpnService API to create a virtual TUN network interface. All traffic on the device gets routed through that interface.

The actual proxy logic lives inside xray or sing-box running as a local process. The chain looks like this:

```
App traffic → Android VpnService TUN interface → tun2socks → local socks5 proxy (xray/sing-box) → encrypted tunnel → your VPN server → internet
```

The local socks5 proxy is the bridge between the TUN interface and the actual xray core. It listens on a port on localhost, typically something in the range of 1080, 10808, or similar, and accepts connections from tun2socks which then forwards all your device traffic through it.

---

## The authentication problem

The local socks5 proxy has no authentication configured. From xray's perspective this makes some sense because the intended client is tun2socks running on the same device, so there was presumably no threat model that included other apps on the same device connecting to it maliciously.

That assumption is now broken.

Because the proxy is unauthenticated and listening on localhost, any process on the device can connect to it. You do not need root. You do not need any special permissions. You just need to know or guess the port, which is trivial since the common defaults are well documented and the RKN methodology document the researcher found explicitly lists the ports to scan.

A malicious app can do this:

```
1. Scan localhost ports 1080, 10808, 9999, and others from the RKN list
2. Attempt a SOCKS5 handshake on each open port
3. If successful, make an outbound connection through the proxy to a known IP echo service
4. The response contains your VPN server's exit IP address
5. Report that IP back to a remote server
```

This requires no special privileges, no root, no exploitation of any kernel vulnerability. It is just a normal network connection from one app to another on the same device.

---

## Why per-app split tunneling does not help

Split tunneling lets you exclude certain apps from the VPN tunnel. You might exclude your banking app thinking this keeps it separated from your VPN setup. It does not protect against this attack.

Split tunneling works at the VpnService routing layer. It controls which apps' traffic goes through the TUN interface and which goes directly to the network. But it has no effect on local network communication. An excluded app can still open a local socket and connect to localhost ports. The VpnService boundary only applies to traffic leaving the device, not to communication between processes on the device itself.

---

## Why Knox, Shelter, and Island do not help

These tools create isolated app environments sometimes called work profiles or private spaces. Apps inside them cannot see apps outside them, cannot access each other's files, and run in separate user contexts.

The isolation breaks down at the loopback interface.

Android's loopback interface (127.0.0.1) is a virtual network interface that exists at the kernel level and is shared across all user profiles on a device. When xray binds its socks5 proxy to 127.0.0.1:10808 it is binding to a resource that is visible to every user context on the device, including sandboxed profiles created by Knox or Shelter.

An app inside a Shelter profile can open a socket, connect to 127.0.0.1:10808, and successfully communicate with xray running in the main profile. The filesystem is isolated. The apps are isolated. The loopback is not.

This is the specific architectural gap that GrapheneOS addresses by giving each profile its own separate network namespace with its own loopback interface.

---

## The Happ situation in detail

Happ has an additional vulnerability beyond the socks5 auth issue. They expose the xray API without authentication, specifically with HandlerService enabled.

The xray API is a management interface that lets external processes control and query the running xray instance. With HandlerService enabled it is possible to:

Query the running configuration and retrieve the full config including server addresses, ports, user IDs, and keys. List active inbound and outbound handlers. Retrieve connection statistics and active session information.

Combining the leaked config with a known class of xray configuration vulnerability (which the researcher deliberately did not describe publicly) could allow an attacker to decrypt traffic. The researcher confirmed this is possible in certain common configurations but withheld the specifics to limit immediate exploitation.

The only safe response to Happ is to block it at the server level by UserAgent and stop using it entirely.

---

## Why UDP breaks socks5 authentication

The correct fix for the unauthenticated socks5 proxy is to require authentication before accepting connections. This is straightforward to implement and several developers have committed to doing it.

The complication is UDP.

In the socks5 protocol, UDP traffic is handled through a UDP ASSOCIATE command. When a client successfully authenticates with username and password and requests UDP forwarding, the server opens a UDP relay port and tells the client where to send UDP datagrams. The problem is that subsequent UDP datagrams are sent directly to that relay port without any per-packet authentication. The auth happens once at session establishment but the UDP relay port itself accepts packets from anyone who knows its address.

On a local loopback this means that even with socks5 auth enabled a malicious app could observe the UDP relay port being opened and send packets directly to it. This does not expose the VPN server IP the same way as the TCP attack but it does create a bypass path.

The practical fix is to disable UDP through the socks5 proxy when auth is enabled. UDP can potentially be handled through alternative mechanisms but in the short term the choice is between authenticated TCP only or unauthenticated TCP and UDP.

---

## What a correct fix looks like

In xray config, enabling socks5 authentication looks like this:

```json
{
  "inbounds": [
    {
      "protocol": "socks",
      "settings": {
        "auth": "password",
        "accounts": [
          {
            "user": "localuser",
            "pass": "randomlygeneratedpassword"
          }
        ],
        "udp": false
      }
    }
  ]
}
```

The password does not need to be secure in the traditional sense since it never leaves the device. It just needs to be something a scanning app cannot guess without having read the config file, which requires filesystem access that the sandbox does properly isolate.

The client application also needs to be updated to pass credentials when connecting to its own local socks5 proxy, which is why this is a code change that needs to ship in an app update rather than something users can configure themselves in most clients.
