![Visitor Count](https://visitor-badge.laobi.icu/badge?page_id=use3r-riddl3r.vless-vuln-awareness)
# VLESS Vulnerability Awareness

This repo exists because a Russian security researcher published a critical finding about nearly every popular VLESS/xray client, asked the community to spread the word, and most people outside of Russian-speaking circles still haven't heard about it. This is an English language summary with practical advice on what to do about it.

This is purely defensive research information. 

---

## Why this matters beyond Russia

Before getting into the technical details it is worth stepping back and thinking about what this vulnerability actually represents in a broader context.

The internet was built as a decentralized, open system. Over time governments, corporations, and intelligence agencies have worked steadily to erode that openness through surveillance infrastructure, app store control, mandatory data retention laws, and now what this article describes: forcing private companies to embed state spyware directly into consumer software(allegedly).

This is not unique to Russia. Similar dynamics exist in China, Iran, and increasingly in countries that would not describe themselves as authoritarian. The tools and techniques travel. What gets deployed against dissidents and journalists in one country gets studied and copied elsewhere. The fact that Russian banking apps are now expected to help identify VPN users is a glimpse at a direction of travel, not an isolated edge case.

Be skeptical of influencers, YouTube channels, and social media accounts that heavily promote specific VPN products, especially commercial ones ( i know most are aware of this but common sense is not so common in 2026 ). VPN marketing is one of the most affiliate-commission-driven spaces on the internet. Many people recommending products have a direct financial incentive to do so and no particular reason to tell you about the problems. The security community that actually finds and discloses vulnerabilities like this one operates very differently, usually without monetization, and usually asking only that you share the information.

The researcher who found this vulnerability did not sell it. They reported it to the developers, got ignored by most of them ( as normal in these cases :/ ), and then published it publicly so that ordinary people would have a chance to protect themselves before state-level actors exploited it at scale. 

Open source software is one of the few meaningful defenses we have. When code is open you can read it, audit it, fork it, and verify that it does what it claims. When it is closed you are trusting a company that may be under legal pressure you will never hear about. The clients affected by this vulnerability are all open source, which is precisely why the researcher could find and document the flaw in the first place. The fix will also come from open source. That transparency is the whole point of this repo existing.

---

## What's going on

Its claimed from multiple sources that Russia's Ministry of Digital Affairs (Минцифры) has been pressuring accredited Russian IT companies to embed spyware modules into their apps (allegedly). The goal of these modules is to detect and report personal VPN servers so they can be blocked. Think of it as a tiny Roskomnadzor agent sitting inside every Russian banking app, Yandex product, and government service on your phone.

The researcher found a vulnerability that makes this threat much more dangerous than it would otherwise be.

---

## The actual vulnerability

Every popular VLESS client based on xray or sing-box works roughly like this:

```
VpnService → tun2socks → xray socks5 → your VPN server → internet
```

The problem is that the local socks5 proxy xray creates has **zero authentication**. Any app on your device can connect directly to it on localhost, completely bypassing the VPN tunnel, and immediately find out the real exit IP address of your VPN server.

Android private spaces like Knox, Shelter, and Island make this worse because while they do isolate the VpnService, **they do not isolate the loopback interface**. That means spyware running inside a sandboxed profile can still scan localhost ports, find the socks5 proxy, and leak your server IP. You might think you are protected when you are not. This is a false sense of security that could get people in genuinely dangerous situations hurt.

The researcher also noted that Russia's censorship agency published an internal document listing exactly which ports to scan for, which strongly suggests they already know about or are very close to discovering this technique themselves.

---

## Who is actually at risk

You are at risk if you have any of the following installed on the same device as your VLESS client: Russian government apps like Gosuslugi, Russian banking apps like Sberbank or Tinkoff, Yandex apps of any kind, or any app from a Russian accredited IT company.

If none of those are on your device you are largely fine. The unauthenticated socks5 port is only dangerous if something is already there to probe it. But given how normalized it has become for people to install banking apps, government service apps, and big tech products alongside privacy tools, the realistic attack surface is large.

---

## Affected clients

The researcher tested and notified all of these on March 10 2026. As of April 7 2026 none had shipped a fix.

| Client | Status |
|---|---|
| Happ | Critically vulnerable, extra dangerous, see below |
| v2RayTun | Vulnerable |
| V2BOX | Vulnerable |
| v2rayNG | Vulnerable |
| Hiddify | Vulnerable |
| Exclave | Vulnerable |
| Npv Tunnel | Vulnerable |
| Neko Box | Vulnerable |
| Clash (common configs) | Vulnerable |
| sing-box (common configs) | Vulnerable |
| Throne (formerly Nekoray, desktop) | Likely vulnerable, same sing-box architecture, not explicitly tested |

---

## Happ is especially bad

Happ deserves its own section. On top of the socks5 issue, Happ also exposes the xray API with no authentication, including HandlerService. This lets any app on the device dump your full VPN config including server IP, keys, and SNI. Combined with another xray misconfiguration the researcher declined to name publicly, it could allow decryption of your traffic entirely.

When the researcher reported this to Happ the developers dismissed it, claimed it was needed for connection statistics (which the researcher demonstrated was false), and initially refused to fix it even under pressure from their own users. Happ was eventually removed from the Russian App Store so they cannot push a fix even if they wanted to.

The researcher's recommendation is to block Happ on your VPN server by UserAgent (`Happ/*`) because even a single user connecting with Happ can expose your server's entry IP and SNI to a censor.

**Do not use Happ under any circumstances.**

---

## What you can do right now

**Best option for most people: separate device for sensitive apps**

Keep a cheap spare Android phone for banking and government apps only. Never install a VPN client on it. Simple, clean, no mixing. This completely eliminates the threat because the spyware and the VPN client never share a device or a network namespace.

**Best single device option: GrapheneOS with profiles**

This is where GrapheneOS genuinely solves the problem in a way that stock Android and third party sandboxing tools simply do not.

Stock Android private spaces like Knox, Shelter, and Island isolate apps at the application layer but they share the same loopback interface. Loopback is the internal 127.0.0.1 network your device uses to communicate with itself locally. Because it is shared across profiles in stock Android, an app in a sandboxed profile can still reach out to localhost, scan ports, and find the unprotected socks5 proxy running in your main profile. The sandbox looks solid but has a fundamental hole in it at the network level.

GrapheneOS fixes this properly. Each user profile gets a fully isolated environment. That means a separate filesystem so no profile can read another profile's files, configs, or keys. Separate app installs so what is installed in one profile does not exist in another. Separate storage so photos, downloads, and documents are completely partitioned. And critically, a separate network namespace including its own isolated loopback interface.

That last part is what matters here. An app running in a secondary GrapheneOS profile is completely blind to what is running on localhost in the owner profile. The loopback is not shared. The spyware in your banking profile cannot reach the socks5 proxy in your VPN profile because from its perspective that proxy does not exist. They are running in fully separate environments that happen to share the same physical hardware.

A practical GrapheneOS setup for this threat:

Owner profile holds your VLESS client and nothing from any company that might be under legal pressure to surveil you. Ever.

Secondary profile holds your banking apps, government apps, and anything else you need but do not fully trust. No VPN client in this profile.

Because the filesystem is also isolated, even if something in the secondary profile tried to read your VPN configs or keys it would find nothing. The separation is complete at every layer that matters.

You need a supported Pixel device to run GrapheneOS. Installation is well documented at grapheneos.org and is straightforward if you are comfortable with basic device setup.

**Server side mitigations**

Set up separate entry and exit IPs on your server. If the censor learns your exit IP they can block it but you can still connect through the entry IP. If a second IP is not possible, wrapping your exit traffic through Cloudflare WARP achieves a similar result since the IP that gets exposed would be a Cloudflare IP rather than your personal server.

Use split routing so Russian IP ranges go direct and everything else goes through your proxy. Then on the server side block routing back to Russian IPs to prevent traffic pattern analysis from services like Yandex identifying that you are using a proxy.

On Windows, v2rayN already has a preset for this called "All except Russia".

**Wait for client fixes**

The root fix is enabling socks5 authentication so even if something finds the port it cannot use it. The tradeoff is that UDP has to be disabled since UDP in socks5 effectively bypasses auth anyway. Some developers have promised fixes. Track the issue trackers of whichever client you use.

---

## Using a second VPN as a tunnel

If you connect to your xray server through a commercial VPN first, the exposed IP would be the commercial VPN's IP rather than your personal server. This is better than nothing since commercial VPN IPs are bulk and harder to act on individually, but it does not fix the underlying issue. Spyware can still find the socks5 port and probe it, it just gets a less useful IP back.

---

## A note on trust and the open source ecosystem

The fact that this vulnerability was found, documented, and published at all is a direct result of open source software. The researcher could read the xray and sing-box code, understand exactly what the local socks5 proxy was doing and why it had no authentication, build a proof of concept, and publish everything for anyone to verify independently.

Closed source VPN clients give you none of that. You cannot audit them. You cannot verify their claims (don't trust no logs marketing). You cannot know whether they have been compelled to add a module similar to what multiple government has been demanding from domestic companies. Several commercial VPN providers have been caught logging traffic they claimed not to log, handing data to law enforcement, or operating in jurisdictions where they are legally required to cooperate with state requests regardless of what their marketing says.

The developers who responded well to this disclosure, thanked the researcher and committed to fixing the issue, are the ones building in public with their reputations and code on the line. The ones who dismissed it or went quiet are the ones you should think carefully about trusting with your safety.

Internet freedom is not a technical problem with a permanent technical solution. It is an ongoing contest between people who want to communicate freely and institutions that want to monitor and control that communication. Open source tools, transparent security research, and communities that share information like this repo is trying to do are part of how people who value freedom stay ahead. The moment we stop paying attention, stop auditing, and stop sharing is the moment the tools we rely on quietly stop working for us.

---

## Credits and sources

All credit for discovering and disclosing this vulnerability goes to the original researcher who published on Habr. They explicitly asked the community to help spread this information, which is why this repo exists.

Original Habr article : https://habr.com/ru/articles/1020080/

POC for per-app split tunneling bypass: https://github.com/runetfreedom/per-app-split-bypass-poc  

Advanced VPN detector by @cherepavel: https://github.com/cherepavel/VPN-Detector  

GrapheneOS profiles documentation: https://grapheneos.org/features  

Throne (formerly Nekoray): https://github.com/throneproj/Throne  

---

## Patch tracker

This section will be updated (will try my best) as clients ship fixes. PRs to update this table are welcome :))


| Client | Fixed | Notes |
|---|---|---|
| Happ | Agreed to fix after community pressure | Cannot ship update, removed from store |
| v2RayTun | Promised fix | Pending |
| v2rayNG | No response | Pending |
| Hiddify | No response | Pending |
| Throne | Not tested in original disclosure | Pending |
| All others | No response | Pending |

![Hiddify](https://img.shields.io/github/v/release/hiddify/hiddify-app?label=Hiddify%20Latest&color=red)
![v2rayNG](https://img.shields.io/github/v/release/2dust/v2rayNG?label=v2rayNG%20Latest&color=red)
![Throne](https://img.shields.io/github/v/release/throneproj/Throne?label=Throne%20Latest&color=red)
![Hiddify](https://img.shields.io/github/issues/search?q=repo%3Ahiddify%2Fhiddify-app+socks5+auth&label=socks5+fix+issue)

![Last Updated](https://img.shields.io/github/last-commit/use3r-riddl3r/vless-vuln-awareness?label=last%20updated)
