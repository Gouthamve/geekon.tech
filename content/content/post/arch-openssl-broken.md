+++
title = "When Arch broke: Grappling with a broken OpenSSL upgrade"
date = "2017-05-01T02:48:15+05:30"

socialsharing = true
totop = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["linux"]
+++

## Introduction

I heard two things about Arch:

1. Its the most awesome distro eva!
2. Everything is bleeding edge and upgrades break dependencies frequently. You need to be ready to spend hours fixing it.

And both turned out to be true for me. For >8months, Arch was awesome and perfect, but quite recently it broke big. I am not going to go into the awesome part now but rather into the time when arch broke spectacularly. A bad upgrade decision broke the OpenSSL installation on my system and caused this:
<img width="100%" src="/img/posts/seven/IMG_20170430_173333.jpg"/>

Now anybody after basic googling will tell me that I need to run `sudo pacman -Syu` to fix my system but I broke both `sudo` and `pacman`! This is the story of how I fixed this.

## Prologue

Everything started with my Networks assignment: "Write a DNS Client and DNS Proxy/Cache server w/o any libraries. Model the client after nslookup". And arch doesn't come with `nslookup`, it lives in a package called: `dnsutils`. So I set out to install it. But before we install anything it is recommeded that I do a update of the existing packages first via: `sudo pacman -Syu`. Now given how Arch releases updates and how big they can get, not running for a couple of weeks (I had exams!) means you might have to download `>1GB` to run an update. I did not have the time (>1hr on my network), so I ran: `sudo pacman -S dnsutils` to install nslookup. 

But it depends on a newer version on OpenSSL and without much thought I update OpenSSL with: `sudo pacman -S openssl`. Thats where I shot myself in the foot! It install openssl-1.1 and removed openssl-1.0. And boom everything broke, well, almost everything. I couldn't run sudo, ping and I thought I was done. I had a frigging assignment close to the deadline! Luckily Chromium must have a vendored version of OpenSSL and I could still work on my assignment. It also did not hurt that `nslookup` was working though ;)

## The Fix

### Diagnosis
A quick look at the errors thrown show that I was missing two files `libssl.so.1.0.0` and `libcrypto.so.1.0.0` And some googling for the errors showed that I should have let the system update finish before I installed a new version. I also came to know running `sudo pacman -Syu` would fix this. But given that both `sudo` and `pacman` are broken I was out of ideas on how to proceed.

I posted on Facebook asking for help and as always the biggest Linux expert on my network, [Ajay Brahmakshatriya](https://github.com/AjayBrahmakshatriya), had an answer. I just had to grab `libssl.so.1.0.0` and `libcrypto.so.1.0.0` from somewhere and put them in the right places. Quoting him:
> Try getting an old version of libcrypto.so and placing in the same directory as pacman and sudo. Since these are shared libraries, the lookup path starts with same directory and the goes to look into other folders.

I decided to strace just to figure out the directories (?!?!) of pacman and sudo. But I was surprised to see that they lookup on absolute paths.

```
open("/usr/lib/tls/x86_64/libcrypto.so.1.0.0", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib/tls/x86_64", 0x7ffe6b058e20) = -1 ENOENT (No such file or directory)
open("/usr/lib/tls/libcrypto.so.1.0.0", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib/tls", 0x7ffe6b058e20) = -1 ENOENT (No such file or directory)
open("/usr/lib/x86_64/libcrypto.so.1.0.0", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib/x86_64", 0x7ffe6b058e20) = -1 ENOENT (No such file or directory)
open("/usr/lib/libcrypto.so.1.0.0", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
```

### Getting the files
Now that I know where to put them, I now needed the files themselves. Like a complete idiot, I decided to compile them! I downloaded the latest OpenSSL release and compiled it. Well, turns out I have the latest release installed (thats how I broke everything, remember?) and its a `1.1` release. It produced `.so.1.1` files! Lesson learnt, I grabbed the last `1.0.0` release. At this point, I should mention that there was a `1.0.2` release but I expected to produce `.so.1.0.2` files.

I compiled it. But the compilation did **NOT** produce any `.so.1.0.0` files. It produced only `.a` and `.pc` files. I dug through the `Makefile` and  explicitly invoked `make libcrypto.so.1.0.0`. Now that produced a whole ton of errors:
```
.
.
.
/usr/bin/ld: libcrypto.a(ts_req_utils.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libcrypto.a(ts_req_print.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libcrypto.a(ts_rsp_utils.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libcrypto.a(ts_rsp_print.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libcrypto.a(ts_rsp_sign.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: libcrypto.a(ts_rsp_verify.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
.
.
.
``` 
Well luckily for me, it clearly told me to add the `-fPIC` flag and adding it to the `CFLAGS` ended up producing the `.so.1.0.0` files. Well, I got the files, but `/usr/lib` is not writable without `sudo`. Oh FML! 

### Bootable USB and chroot
I luckily remembered the time when I fucked up the perms on all binaries in `/usr/bin` but fixed it by burning a bootable and using `arch-chroot` to chroot into the current installation. I downloaded the latest image, burnt a USB and booted it up. Buuut, luck was not in my favour and I ended up in some sort of recovery shell with `SQUASHFS errors`. I thought it was a faulty USB and burnt it on another USB. Same error :( Luckily someone on the internet asked to check the file checksum as the file could have been corrupted over the network, and indeed, I found that the checksums did not match. Lesson: **Always verify the file checksums!!**

Now I had good USB drive and chrooted via:
```bash
mount /dev/sda1 /mnt
arch-chroot /mnt
```

I copied the files to `/usr/lib` and tried sudo, it worked!! Then I tried `pacman`, it did NOT!! It was missing some symbols like `EVP_aes_128_ctr`. Oh fuck! 

### OpenSSL version matters. A lot!
I pulled the wrong version down! But now that I got `sudo` working, I exited the chroot and USB-env and booted in the main system. I now figured that I would have to check the `1.0.2` series and downloaded the last release in that series. Given this is my 3rd version to be compiled, I was losing some patience. Compilation still did not produce any `.so` files but I fixed it using the same steps used on the `1.0.0` version.

I was hoping for it, but I was more than a little surprised to see that the `1.0.2` version produces `.so.1.0.0` files! I put them in the right places and boom, everything was working. I ran `sudo pacman -Syu` and >30mins later pacman downloads all the dependecies. Buuuut, it fails an integrity check as `/usr/lib/libssl.so.1.0.0` and `/usr/lib/libcrypto.so.1.0.0` already existed (duh!). I looked at the lookup path again and moved the files to `/usr/lib/tls` and `pacman` succeeded this time! I finally had legitimate files in `/usr/lib` I deleted the `/usr/lib/tls` folder and am now enjoying my system :P

## Epilogue
My system is back and its beautiful  <3 

But I did learn a couple of things from this experience:

1. Arch can break but everything can be fixed. Just ask Ajay!
2. Only update something after you do a full system update.
3. Do full system updates frequently! Daily! Figure out how to enforce it!
4. Always have a backup live USB ready.

Also in retrospect I should have just nicked the files off the bootable USB I burnt (do they exist there?).


## Detour: Why Arch?
Post my awesome internship last summer, I decided to invest the stipend money from there into a workstation. When the workstation finally ended up in my hands, like any linux beginner, I decided to install Ubuntu.  But I was a heavy user with 2 monitors and 4 workspaces and I soon observed that some process was eating up a ton of my RAM (**800MB** when idle!) and CPU. Some very basic debugging with `htop` showed it was compton from Ubuntu's window manager.

Even though I had a lot of RAM to burn, I decided to investigate "lightweight" options and loved the screenshots of an Arch system with i3wm.  I wanted to get better at managing Linux systems and decided to give Arch a shot and spent a full night up getting a decent Arch system with i3wm working. And it was probably one of my best decisions. I am pretty sure it made me better at fixing things and drastically improved my terminal-fu.

Psst, i3 rules with a <10MB memory usage!