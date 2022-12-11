---
title: Arsenal & HTTPS Inspections
categories: [mobile,reverse]
tags: [android,apk,https,frida,adb,root]
---

Hello, i'll show you how I setup a https inspections environment for reveal private api.
Let's go


__TODO LIST__
- [x] rooted Android
    + [x] anti root
    + [x] system-wide cert
    + [x] rooted proxy
- [x] frida server (obfuscated 👍)
- [x] handle ssl pinning (.js script)
- [x] mitm proxy
- [x] apktool
- [x] ADB 
- [x] pc (win or linux)

___

## Hardware

I use an not so old samsung device that i've root, and config several tools like anti root to prevent application checks.

We'll need 
: the cert of the mitm proxy server to allow the network inspection, and a rooted proxy app to redirect requests on my pc.

It's a classic man-in-the-middle attack with ssl consideration. 

## SSL 

To be able to read https requests, I'll present 2 different ways but be **carefull**

_it may not working all the time depending on the context_ :

**(Static way)** 
- decompile the apk
- patch it for https inspections 
- rebuild it and install to device

**(Dynamic way)**
- frida (instrumentation tool kit)
  - with multi-bypass ssl pinning script


> And sometime you'll need app's client certificate config to the rooted proxy on the device. 
{: .prompt-tip }

## Static way

Download the targeted app on [apkcombo.com](https://apkcombo.com) or alternative one.

1. Decompile
: We'll using apktool
```console
$apktool d file.apk
```
2. Path app
: I use apk-mitm from @shroudedcode for patching and signing apk
```console
$npm install -g apk-mitm
$apk-mitm file.apk
```
3. Rebuild
: ddd
```console
$apktool b file.apk
```
At this point, we've a patched and signed apk if there's no error during the process.

We can try to run it on my rooted device if it don't crash at all, and **waiting for the next step**.

## Dynamic way

> Frida
: Is a dynamic instrumentation toolkit for developers, reverse-engineers, and security researchers.
{: .prompt-info }

Here I'll not share my used scripts. Simply show you some tips by using frida.

ADB platform
: [Windows](https://dl.google.com/android/repository/platform-tools-latest-windows.zip), [Mac](https://dl.google.com/android/repository/platform-tools-latest-darwin.zip) or [Linux](https://dl.google.com/android/repository/platform-tools-latest-linux.zip)

Don't forget to install [Frida](https://frida.re/docs/quickstart/)


Now, we'll need to download the [frida server binary](https://github.com/frida/frida/releases) with the good architecture for your device.

Using **adb** we'll push the binary on the device (usb debug enabled).

`list attached devices (usb)`
: ```console
$adb devices
```

`open device shell and get root right`
: ```console
$adb shell
$adb su
```

`(be carefull root) push file on device and run exemple`
: ```console
#adb push source destination
#/data/local/tmp/frida-server &
#exit
```

Ok, your device has not crashed and the frida server is **running**.

## almost at the end

On pc side, prepare [mitm proxy](https://mitmproxy.org/) and run a webproxy listener.

```console
$mitmweb --listen-port 8999
```

On device side, run rooted proxy app configured for my pc.


Now here we can choose 2 communication methods 
- USB cable (debug)
- WIFI (more stealth) 

For the exemple here I'll show you with the USB method.

```console
$frida -U -f app_id -l bypass_script.js --no-pause
```

bypass_script.js
: here a [community code share platform](https://codeshare.frida.re/)

After runned frida cli this will spawn the app from the frida server with hooking script.

That's all folk !

For conclusion, we use a command that call an agent on the device that run the targeted app bypassing ssl using frida .js script and all decrypted traffic go through my webproxy listener on the pc.

![](https://mitmproxy.org/mitmweb.png)