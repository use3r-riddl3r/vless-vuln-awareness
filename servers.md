# Server Hardening

This guide covers the server side mitigations that complement the device level protections described elsewhere in this repo. Even if a censor learns your VPN server's exit IP through the socks5 vulnerability, a well configured server can limit the damage and keep you connected.

---

## Block Happ immediately

This is the most urgent action if you run a server that other people connect to.

Happ exposes the full xray config including your server's entry IP and SNI to any app that probes the xray API on a user's device. Even a single user with Happ installed can hand your server over to a censor. You have no visibility into what clients your users are running.

Block Happ connections at the server level by UserAgent. In your xray or sing-box server config, add a rule that rejects connections with a UserAgent matching `Happ/*`.

In 3x-ui or similar panels there is usually a field to block specific UserAgent strings in the inbound settings. If you are managing raw xray config directly the approach depends on your transport. For VLESS over WebSocket you can add a header filter at the reverse proxy level (nginx or caddy) that drops requests with the Happ UserAgent before they reach xray.

A basic nginx example:

```nginx
if ($http_user_agent ~* "Happ/") {
    return 403;
}
```

This goes in the location block that proxies to your xray websocket.

---

## Separate entry and exit IPs

The most robust structural defense is having different IPs for accepting connections and for sending traffic to the internet.

When a censor probes a device running Happ or exploits the socks5 vulnerability they learn your server's exit IP, the IP that appears when your users browse the web. If your entry IP (the one clients connect to) is different from your exit IP, knowing the exit IP does not help them block connections. Users can keep connecting through the entry IP even if the exit IP gets blocked or flagged.

Most VPS providers allow you to attach multiple IP addresses to a server. Add a second IP, configure xray to accept inbound connections on the first IP, and route all outbound traffic through the second IP. The exact method depends on your OS and provider but in general it involves adding the second IP to your network interface and then setting the default route for outbound traffic to use it.

A simpler alternative that achieves a similar result is routing your exit traffic through Cloudflare WARP. Install the WARP client on your server and configure xray to send outbound traffic through it. Your exit IP becomes a Cloudflare IP. Cloudflare IPs are used by millions of people and are much harder to act on than a personal server IP. WARP has a free tier that is sufficient for this purpose.

---

## Split routing

Split routing means sending Russian IP ranges direct without going through your proxy, and routing everything else through the tunnel. This matters for two reasons.

First, Russian services often work faster and more reliably when accessed directly rather than through a proxy. Practically speaking your users will have a better experience.

Second and more importantly, if your users' traffic to Russian services goes through your server it becomes visible to those services. Yandex, VK, and similar platforms can analyze traffic patterns and identify that connections are coming from a proxy server. They can report this to RKN or it can be used in traffic correlation attacks to identify your server even without the socks5 vulnerability.

On the client side, v2rayN on Windows has a built in preset called "All except Russia" that handles this. For other clients you need to add routing rules that direct geoip:ru traffic to direct and everything else to the proxy outbound.

On the server side, block outbound connections back to Russian IP ranges entirely. This stops your server from being used as a relay back into Russian services, which eliminates the traffic analysis risk from that direction.

In xray server config this looks like adding an outbound rule that blocks geoip:ru:

```json
{
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom"
    },
    {
      "tag": "block",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "outboundTag": "block",
        "ip": ["geoip:ru"]
      }
    ]
  }
}
```

This rejects any traffic your server tries to send to Russian IP ranges. Adjust the geoip tag based on your situation. Iran users would add geoip:ir. Chinese users geoip:cn.

---

## Keep your domain and SNI clean

Your server's domain name and SNI are as important as its IP address. If Happ leaks your SNI to a censor they can block the domain even if they cannot block the IP, or use it to identify your server across IP changes.

A few practices that help here. Use a domain that is not registered to your real identity and paid for with privacy-respecting methods. Do not reuse domains across servers. Do not point your server domain at anything else that could be linked back to you. Consider using Cloudflare proxying for your domain so that DNS lookups return Cloudflare IPs rather than your server IP directly, giving you another layer of separation.

---

## Rate limiting and connection monitoring

If your server is being actively probed or your IP has been identified you will sometimes see unusual connection patterns. Setting up basic monitoring helps you notice when this is happening.

Log connection attempts and look for patterns like many failed connections from Russian IP ranges, repeated connection attempts with unusual UserAgent strings, or connection patterns that do not match your known users' behavior.

Basic fail2ban rules can automatically block IPs that make too many failed connection attempts. This is not a perfect defense but it raises the cost of automated scanning.

---

## Updating regularly

xray and sing-box both release updates that address security issues and improve detection resistance. Keep your server side core up to date. If you use a panel like 3x-ui it typically handles core updates through its interface. If you manage xray directly, subscribe to the release notifications on GitHub and update when new versions drop.

The same applies to your underlying OS. A server running an unpatched kernel or with known vulnerabilities in other services is a much larger risk than the socks5 issue.

---

## If your server IP gets blocked

Have a plan before it happens. Know which provider you are using and how quickly you can spin up a new instance. Have your xray config backed up somewhere secure so you can restore it quickly on a new server. Consider having a secondary server in a different datacenter and country as a backup that users can switch to.

The goal is to make blocking your server an inconvenience rather than a crisis. If you can be back up with a new IP in under an hour the practical impact of exposure is manageable.
