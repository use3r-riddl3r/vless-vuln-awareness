# Threat Model

Not everyone reading this is in the same situation. A journalist in Moscow using VLESS to file stories faces a completely different risk level than someone in Germany using it to access streaming content. This document helps you think through your own situation honestly so you can apply the right level of protection without either panicking unnecessarily or being complacent about real risk.

---

## The core question

The vulnerability described in this repo requires two things to be true at the same time. You need to be running a VLESS client. And you need to have an app on the same device that is actively trying to probe your localhost socks5 proxy and report what it finds.

If only the first thing is true you are not currently at risk from this specific attack. If both are true the risk is real and potentially serious depending on what you are using VPN for and where you are.

---

## Who is most at risk

**People in Russia using VLESS to access blocked content and who have Russian banking or government apps on the same device.** This is the primary threat scenario the researcher was describing. Sberbank, Tinkoff, Gosuslugi, VTB, and similar apps are from companies that are either state controlled or operating under strong state pressure. The researcher's findings about government methodology documents suggest RKN is actively working toward exploiting exactly this attack surface.

**People in Russia who run VPN servers for others.** Even if your own device is clean, the researcher specifically notes that a single Happ user connecting to your server exposes your server's entry IP and SNI. If you operate a server with multiple users you are partly dependent on their clients not being Happ.

**People in Iran using similar tooling.** The Iranian censorship apparatus is sophisticated and has historically been an early adopter of techniques that Russia later implements. The same apps, the same protocols, and the same attack surface apply.

**Activists, journalists, and lawyers in any country where their work creates state-level adversaries.** The techniques in this document travel across borders and get adopted by state actors beyond Russia and Iran. If your work puts you in conflict with a government that has the motivation and capability to target you specifically, the bar for protection should be higher than for casual VPN users.

---

## Who is at lower risk from this specific attack

**People outside Russia and Iran using VLESS for general privacy or content access who do not have apps from affected companies installed.** If your device has no Russian or Chinese state-adjacent apps on it there is nothing present to probe the socks5 port. The vulnerability exists but there is no exploiting agent.

**People using VLESS on desktop systems they control.** The attack requires malicious software already running on the device. On a Linux desktop where you control what is installed and are not running banking apps the practical risk is much lower. Throne on Arch Linux with no untrusted software installed is a very different situation from an Android phone with ten banking apps.

**People using WireGuard rather than VLESS.** WireGuard does not use a local socks5 proxy in the same way. The specific attack vector described here does not apply to WireGuard based setups.

---

## Thinking about your specific situation

Ask yourself these questions honestly.

What apps do you have installed on the same device as your VPN client? Go through your app list and think about which companies built them and what jurisdiction those companies operate under. Banking apps from major institutions in countries with strong surveillance laws are a concern. Government apps are a concern. Apps from large tech companies with known state relationships are a concern.

What are you using VPN for? Accessing YouTube and Netflix is a different risk profile from communicating with sources, organizing politically, or circumventing surveillance that is specifically directed at you. The more sensitive your use case the more seriously you should take this.

What would happen if your VPN server IP got exposed? If it is a shared commercial VPN the answer is not much, that IP is already known. If it is your personal server that only you use, exposure means your connection method gets blocked. If it is a server that other at-risk people depend on, exposure has consequences for them too.

Are you a target or are you in a population? Mass surveillance systems that scan for VPN usage treat everyone similarly. Targeted surveillance against specific individuals involves more sophisticated and persistent methods. Most people are in the population bucket, which means the automated techniques described here are the primary concern.

---

## Risk levels and appropriate responses

**Low risk:** You are outside Russia and Iran, you have no state-adjacent apps on your device, and you are using VPN for general privacy or content access. The main thing to do is avoid Happ, stay aware of when client patches land, and keep your setup clean.

**Medium risk:** You are in Russia or Iran, you have some banking or government apps on your device, and you are using VPN to access blocked content rather than for sensitive communications. Switch to a separate device for banking apps, or set up GrapheneOS profiles. Apply the server side mitigations. Stop using Happ immediately if you ever were.

**High risk:** You are in a country with an active surveillance program directed at people doing what you do, you are a journalist, activist, lawyer, or organizer, or you have specific reason to believe you may be a target of state surveillance. Everything in this repo applies with urgency. Separate devices for sensitive apps, GrapheneOS for your primary device, server side hardening, and ideally consultation with a digital security organization that specializes in working with at-risk people. EFF, Access Now, and similar organizations have staff who do this work.

---

## What this vulnerability does not protect against

It is worth being honest about the limits of the mitigations described here.

Fixing the socks5 auth issue protects your VPN server IP from being leaked by apps on your device. It does not prevent your network provider from seeing that you are generating VPN-like traffic patterns. It does not prevent server-side correlation attacks. It does not protect against physical device seizure and forensic analysis. It does not help if the VPN server itself is compromised or operated by someone you should not trust. There are other solutions to these but wont get into them here.

Security is layered and no single fix addresses all threats. The goal here is to close a specific hole that was open and being actively developed toward exploitation. Closing it is worth doing. It is not the end of the story.
