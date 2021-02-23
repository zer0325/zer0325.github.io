---
layout: post
title: All I want is for QEMU to work :)
---

Nung una akala ko a simple `$ pacman -S qemu` is enough. Actually the
installation goes smoothly. No problem encountered as seen below.

![qemu installation screenshot](/assets/images/qemu/qemu_install_1.png)
![qemu installation screenshot](/assets/images/qemu/qemu_install_2.png)

Pero nung ni-run ko na, eto ang lumabas na error. (Screenshot below).

`qemu-system-x86_64: error while loading shared libraries: libnettle.so.8:
cannot open shared object file: No such file or directory`

![qemu first run error screenshot](/assets/images/qemu/qemu_first_run_error.png)

Actually, umpisa pa lang yan. May mga susunod pang mga errors. And at this
point, ito lang ang nasa isip ko "All I want is for QEMU to work", hence the
title. Btw, I am running Arch.

> Output of `$ uname -a`
>
> `Linux tungkunglangit 4.20.0-arch1-1-ARCH #1 SMP PREEMPT Mon Dec 24 03:00:40 UTC 2018 x86_64 GNU/Linux`



#### The journey

From this point onward, ilalagay ko na ang details kung paano eventually gumana
ang **qemu** after nang mga errors na naencounter ko. Note that this is not an
"All-in-one solution" but rather just an approach on how I eventually managed to
make **qemu** work. Pwede rin syang mag serve as a reference kung paano
mareresolve yung mga problems after installation based on the error log.

Una, iaddress natin yun unang error sa taas. First na ginawa ko is look for the
version of *libnettle.so* that I have in `/usr/lib/` directory. Then check with
`$ pacman -Qo /usr/lib/libnettle.so.6` kung sino ang "owner" of that file. The
"owner" of the file is an application named `core/nettle`. Since the version
installed in my system is lower, I think it doesn't hurt to upgrade. So,
inupgrade ko sya as shown in the screenshot below.

![core/nettle upgrade](/assets/images/qemu/nettle.png)

Then syempre, run qemu again `$ qemu-system-x86_64`. Then boom, ERROR.

`qemu-system-x86_64: error while loading shared libraries: libnettle.so.6:
cannot open shared object file: No such file or directory`

![libnettle.so.6 error screenshot](/assets/images/qemu/nettle_error.png)

Notice the error `libnettle.so.6`? Una naghahanap sya nang `libnettle.so.8`.
After mainstall ang `libnettle.so.8` ngayon naman hinahanap nya ang lumang
version? Anong kalokohan to? hahahaha.

Since gusto ko talaga mapagana ang **qemu** and naisip ko andito na eh tuloy na
natin to. So what I did is to create a symbolic link para sa `libnettle.so.6`
and link it to the upgraded version.

![libnettle symbolic link](/assets/images/qemu/libnettle_symbolic_link.png)

The reasoning behind is that, since
`libnettle.so.8` is an upgraded version of `libnettle.so.6`, It should be
backward-compatible. Meaning the functions present in `libnettle.so.6` should
be in `libnettle.so.8`. Kase I doubt na yung author nang library will create an
upgrade and at the same time knowingly break the applications that uses the old
version of his library. And besides pwede naman nating ibalik sa dating version
if ever. Now let's run again. ERROR!! (na naman).

`qemu-system-x86_64: error while loading shared libraries: libhogweed.so.4:
cannot open shared object file: No such file or directory`

![libhogweed error](/assets/images/qemu/libhogweed_error.png)

So `$pacman -Qo /usr/lib/libhogweed.so.6` na naman para malaman kung sino ang
"owner" of that file. It turns out, the owner is `core/nettle`. So again create
a symbolic link for `libhogweed.so.4` and link it to its upgraded version. So
run na naman natin. ERROR!!! (see screenshot below)

`qemu-system-x86_64: /usr/lib/libnettle.so.8: version 'NETTLE_6' not found
(required by /usr/lib/libgnutls.so.30)`

`qemu-system-x86_64: /usr/lib/libnettle.so.8: version 'HOGWEED_4' not found
(required by /usr/lib/libgnutls.so.30)`

![nettle and hogweed error](/assets/images/qemu/nettle_hogweed_error.png)

Note na yung error now is specific. I think yung `NETTLE_6` and `HOGWEED_4` are
the old macros defined in the old version of the library and is now not found
in the upgraded version. Most probably the macros now will be `NETTLE_8` and
`HOGWEED_6` in the upgraded version. But take note sa kung ano file ang
nangangailangan nang old macros, it is `usr/lib/libgnutls.so.30`. So for now
iwan muna natin sya, but take note of the file `/usr/lib/libgnutls.so.30`. And
also, I want to check kung yung `qemu` needs `NETTLE_8` or `NETTLE_6`. So ang
ginawa ko is to return to the old version of `core/nettle` via `$pacman -U
nettle-3.4.1-1-x86_64.pkg.tar.xz`. Note that the file is in
`/var/cache/pacman/pkg/` directory. Also kailangan muna natin i-remove yung old
symbolic link because based on experience hindi magtutuloy ang installation.

![installtion of old nettle](/assets/images/qemu/old_nettle_install.png)

After this create tayo nang symbolic link to `libnettle.so.6` para sa
`libnettle.so.8` when we run `qemu`. And true enough `NETTLE_8` nga ang
kailangan nang `qemu` as evidenced by the screenshot below.

![nettle version 8](/assets/images/qemu/nettle_8.png)

#### Back to square one

So balik sa dati tayo. Since now alam natin ang kailangan nang `qemu` is yung
`NETTLE_8` kailangan lang natin iupgrade ulet yung `core/nettle` application
natin. First remove muna natin yung huling symbolic link na ginawa natin. Then
upgrade yung `core/nettle` and then create the symbolic links for
`libnettle.so.6` and `libhogweed.so.4` just like nung ginawa natin sa umpisa.
But this time meron lang tayong iuupgrade na isa pang file. Remember yung
`/usr/lib/libgnutls.so.30` sya yung iuupgrade natin. Now doing a `$pacman -Qo
/usr/lib/libgnutls.so.30`, I found out that the owner of the file is
`core/gnutls`. So I installed `core/gnutls` and then run `qemu` and wala nang
error about `libnettle`.  Syempre meron na namang ibang error.

`qemu-system-i386: symbol lookup error:
/usr/lib/libvirglrenderer_glEGLIimageTargetTexStorageEXT`

So I then check yung version nang `virglrenderer` application installed in my
system and unfortunately I have the newest version. See the screenshot below.

![viglrenderer error](/assets/images/qemu/virglrenderer_error.png)

So at this point, I am stumped. So I check na lang yung information about the
file `virglrenderer` via `$pacman -Qi virglrenderer`. One of the file that it
depends on is `libepoxy` as shown in the screenshot below.

![virglrenderer info](/assets/images/qemu/virglrenderer_info.png)

So I check kung updated na sya. So I upgraded the file and then run
`qemu-system-x86_64` again and another ERROR.

`qemu-system-x86_64: symbol lookup error: qemu-system-x86_64: undefined symbol:
ZSTD_compressStream2`

![zstd error](/assets/images/qemu/zstd_error.png)

A google search for the keyword "compressStream2" produces a result about
"zstd". So again using pacman check natin if we have an updated `zstd`. Since
hindi sya updated, inupdate ko lang sya and then run again
`qemu-system-x86_64`. Again ERROR na naman.

`qemu-system-x86_64: symbol lookup error: qemu-system-x86_64: undefined symbol:
libusb_wrap_sys_device`

![libusb error.png](/assets/images/qemu/libusb_error.png)

Since this is just a "libusb" error, I just check again if I have an updated
version of `libusb`. Since hindi sya updated, I try to install the updated
version of `libusb`, pero this time my error na naman. Complaining about
breaking dependency in libusbx required by libpcap.

![libpcap error](/assets/images/qemu/libpcap_error.png)

So ang ginawa ko lang is update `libpcap` and then update `libusb`. And then
run again `qemu-system-x86_64` and voila no ERROR.

![qemu running](/assets/images/qemu/qemu_running.png)

Full window screenshot:

![qemu full window](/assets/images/qemu/qemu_full.png)


#### Conclusion

Although it took me hours to finally make **qemu** work, I think it is worth
it. Again this is not an "all-in one solution". And I also doubt that If I can
reproduce the same sequence on other machines. But the biggest take-away here
is that, to take note of the error produced because that is the biggest clue in
solving the problem.
