# GrapheneOS Setup Guide

This guide walks through setting up GrapheneOS with isolated profiles to protect your VPN setup from the socks5 vulnerability described in this repo. It assumes you have read the README and understand why this is necessary.

---

## Why GrapheneOS specifically

Most Android security advice points people toward Knox, Shelter, or Island for app isolation. Those tools are fine for some threat models but as explained in the technical document, they share the loopback interface across profiles. That single shared resource is enough for spyware in one profile to probe the socks5 proxy in another.

GrapheneOS implements proper user profile isolation at the kernel level. Each profile gets its own network namespace which includes its own loopback interface. This is not a feature unique to GrapheneOS in theory but it is the only readily available Android build that actually implements it correctly for user profiles.

Beyond the loopback isolation, GrapheneOS also gives you a hardened kernel, a stronger permission model, better exploit mitigations, and no Google services in the base install unless you explicitly add them in a sandboxed container. For a device you are using for serious privacy purposes it is the right foundation.

---

## Supported devices

GrapheneOS only supports Google Pixel devices. This is because Pixel phones have the best support for hardware security features like the Titan M security chip, verified boot with custom keys, and full firmware update support from GrapheneOS.

Supported devices as of 2026 include Pixel 6 and newer. Older Pixels lose support as Google drops security updates for them. Check grapheneos.org/faq for the current supported device list before buying hardware.

A used Pixel 7 or 8 is a solid choice if you are buying specifically for this purpose. They are reasonably priced second hand and will receive GrapheneOS support for years.

---

## Installing GrapheneOS

The installation process is well documented at grapheneos.org/install. The web installer is the easiest path and works from any Chromium based browser.

The high level steps are:

Enable OEM unlocking in developer options on your Pixel. Boot into fastboot mode. Use the web installer at grapheneos.org/install/web to flash the GrapheneOS image. Re-lock the bootloader after installation. This re-enables verified boot with GrapheneOS's own keys, which is important for the security model.

The whole process takes about 20 to 30 minutes. Read the official instructions carefully rather than following third party guides since the official docs are kept up to date.

---

## Setting up profiles

Once GrapheneOS is installed and set up with your owner profile, creating a secondary profile is straightforward.

Go to Settings, then System, then Multiple users. Enable the multiple users option if it is not already on. Add a new user. You can name it something like Banking or Apps or whatever makes sense to you.

Switch into the new profile by pulling down the notification shade and tapping your profile icon, or through the same multiple users menu. The new profile starts completely clean with no apps installed.

---

## What goes in each profile

Owner profile is your primary environment. Install your VLESS client here, Throne if you are on desktop or whichever Android client you use. Install everything you trust. Do not install anything from Russian companies, Chinese companies with government ties, or any app you cannot audit. This profile's loopback is completely invisible to anything in other profiles.

Secondary profile is your isolation zone. Install banking apps here. Install government service apps here. Install anything where you need it to function but do not fully trust it. Do not install any VPN client in this profile. When you are done using these apps switch back to your owner profile.

If you need more separation you can create additional profiles. Some people run three: a clean owner profile, a banking profile, and a general use profile for apps they want but are slightly skeptical of.

---

## Verifying the isolation works

If you want to confirm the loopback isolation is actually working you can test it yourself.

In your owner profile, start your VLESS client and note which port the socks5 proxy is running on. You can usually find this in the client's settings or logs, commonly 10808 or 1080.

Switch to your secondary profile. Install a simple network scanning app or terminal emulator. Try to connect to 127.0.0.1 on that port. On stock Android with Shelter this connection would succeed. On GrapheneOS it should fail with a connection refused error because the loopback interfaces are separate and the port simply does not exist in the secondary profile's network namespace.

---

## Sandboxed Google Play

If you need apps that are only available through Google Play, GrapheneOS supports installing sandboxed Google Play services. This is different from having full Google services on your device. The sandboxed version runs without special system privileges, just like any other app, while still allowing Play Store app installation and in-app purchases.

Install sandboxed Google Play in your secondary profile if you need it for banking or government apps. Do not install it in your owner profile unless you have a specific reason to. The less Google integration in your primary environment the better.

---

## Day to day usage

The profile switching workflow becomes natural quickly. Most of the time you are in your owner profile using your VPN. When you need to check your bank account you switch profiles, do what you need, and switch back. The switch takes a few seconds.

Notifications from secondary profile apps appear in the notification shade and can be tapped to automatically switch into that profile. You do not need to manually switch before checking a banking notification.

One thing to be aware of is that apps in inactive profiles are suspended and do not receive push notifications unless you have enabled background activity for them. For banking apps this usually does not matter since you only open them actively. For anything that needs to receive calls or messages in the background you may need to adjust per-profile settings.

---

## A note on threat model

GrapheneOS with profiles is a very strong mitigation for this specific vulnerability. It is not a complete solution to all privacy concerns.

Your VPN server can still be identified through traffic analysis if you are not careful about split routing. Your network provider can still see that you are using a VPN even if they cannot see the contents. Physical device seizure bypasses everything. 

Use GrapheneOS profiles as one layer in a broader approach, not as a magic solution. The README covers the server side mitigations that complement this device level protection.
