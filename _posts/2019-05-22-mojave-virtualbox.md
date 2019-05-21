---
title: Mojave + VirtualBox
categories: article
tags: dev virtualbox mojave
---

I have finally decided to upgrade my macs to Mojave from High Sierra. For my work machine however, VirtualBox simply stopped working and throwing `NS_ERROR_FAILURE` on startup. Similar to [this](https://stackoverflow.com/questions/52689672/virtualbox-ns-error-failure-0x80004005-macos). Trying to reinstall won't work because the installer will fail at the last step.

I have tried *everything* suggested from disabling Gatekeeper to many many iterations of uninstall/reinstall. Almost every solution suggested clicking `Allow` on the Security & Privacy tab but it never appeared.

What worked for me finally was: [macos - VirtualBox 5.2 Won't Install on Mac OS 10.13 - Ask Different](https://apple.stackexchange.com/a/360123/332949) which boils down to:

```
spctl kext-consent disable
spctl kext-consent add VB5E2TV963
spctl kext-consent enable 
reboot
```

That is basically the `Allow` part which will finally make the install successful. `VB5E2TV963` is the Oracle id that we are allowing.

If you are in similar situation, I hope this will help.
