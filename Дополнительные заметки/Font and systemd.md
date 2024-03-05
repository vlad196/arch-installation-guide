Источники связанные со следующими проблемами
```
окт 08 17:17:30 archlinux systemd-vconsole-setup[200]: /usr/bin/setfont failed with exit status 71.
окт 08 17:17:30 archlinux systemd-vconsole-setup[200]: Setting fonts failed with a "system error", ignoring.
окт 08 17:17:30 archlinux systemd-vconsole-setup[204]: setfont: ERROR kdfontop.c:183 put_font_kdfontop: Unable to load such font with such kernel version
...
окт 08 17:17:30 archlinux systemd[1]: Starting Virtual Console Setup...
окт 08 17:17:30 archlinux systemd-vconsole-setup[260]: setfont: ERROR kdfontop.c:183 put_font_kdfontop: Unable to load such font with such kernel version
окт 08 17:17:30 archlinux systemd-vconsole-setup[258]: /usr/bin/setfont failed with exit status 71.
...
окт 08 17:17:30 archlinux systemd[1]: Starting Virtual Console Setup...
окт 08 17:17:30 archlinux systemd-vconsole-setup[260]: setfont: ERROR kdfontop.c:183 put_font_kdfontop: Unable to load such font with such kernel version
окт 08 17:17:30 archlinux systemd-vconsole-setup[258]: /usr/bin/setfont failed with exit status 71.

```

Источники:
https://archlinux.org.ru/forum/topic/15021/?page=3
https://bugs.archlinux.org/task/79287
https://archslr.wordpress.com/2015/06/22/%D1%80%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9-%D1%8F%D0%B7%D1%8B%D0%BA-%D0%B2-arch-linux/
https://archlinux.org.ru/forum/topic/18988/?page=2


Влияют параметры ядра quiet и splash
Без них ошибок нет. Надо узнать подробнее про это

setfont в binaries, расположение sd-vconsole и сам модуль plymouth в mkinitcpio не влияют