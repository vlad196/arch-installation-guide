
## Если размонтировал диски до того как настроил систему или случилась ошибка, то после этапа с  sshпишем:

```bash
cryptsetup luksOpen /dev/nvme0n1p2 LVM_part
mount -o compress_algorithm=zstd:6,compress_chksum,atgc,gc_merge,lazytime /dev/mapper/systemvg-root /mnt && \
swapon /dev/mapper/systemvg-swap && \
mount --mkdir /dev/nvme0n1p1 /mnt/efi && \
mount --mkdir /dev/sdb /mnt/mnt/sdb && \
arch-chroot /mnt
```

## Подпись efi приложений  с ядрами, если они не подписаны
```bash
sbctl sign-all
```



## Если остались старые ключи от systemd-cryptenroll (они остаются и после переустановки системы), то можно их удалить:
**Проверяем лист привязанных паролей (Keyslots):**
```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```
**Чистим ключи tpm в модуле:**
```bash
sudo tpm2_clear
```
**Отвязываем все ключи:**
```bash
sudo systemd-cryptenroll /dev/nvme0n1p2 --wipe-slot=tpm2
```

## Если нужно удалить все из boot
```bash
rm -f /mnr/efi/EFI/Linux/*
```
## Если панель kde зависла под wayland
```bash
kquitapp5 plasmashell ; /usr/bin/plasmashell &
plasmashell --replace
```
## Копирование всех ценных файлов
```bash
paru -S dosfstools
sudo mount --mkdir /dev/sdc2 /mnt/sdc && \
sudo mkdir /mnt/sdc/backup_keys /mnt/sdc/backup_keys/secureboot  && \
/mnt/sdc/backup_keys/luks && \
sudo cp -r /usr/share/secureboot/keys /mnt/sdc/backup_keys/secureboot && \
sudo cp -r /mnt/sdb/header-nvme0n1p2.img /mnt/sdc/backup_keys/luks
```

## Востановление secure boot из флешки
```bash
mount --mkdir /dev/sde2 /mnt/flash && \
mkdir -p /usr/share/secureboot/keys && \
cp -r /mnt/flash/backup_keys/secureboot/keys /usr/share/secureboot/
```

## Как обновлять правила в USBGUARD (например быстро обновить правила с новой флешкой. Можно сделать alias)
```bash
sudo zsh -c "usbguard generate-policy > /etc/usbguard/rules.conf" && sudo systemctl restart usbguard
```

## Обновление ядра
**При обновлении ядра paru преложит ещё раз посмотреть PKGBUILD. Изменяем его:**
Вариант 1)
```bash
_use_current=y
```
>[!info]
Далее ядро должно само взять из уже существующего конфиги. При изменении каких-либо параметров в ядре, сборщик предложит посмотреть это и поменять

Вариант 2)
```bash
_menunconfig=y
```
>[!Info]
Далее в настройках загружаем уже заранее подготовленный нами конфиг (Перед этим нужно озаботиться созданием этого конфига)
### Аудит безопасности:
#### LYNIS:
**Установка:**
```bash
sudo lynis audit system
```
Запуск
```bash
arch-audit
```

##Добавляем исключения для правил
/etc/lynis/default.prf
skip-test=LYNIS
skip-test=BOOT-5264
skip-test=AUTH-9282
skip-test=AUTH-9286
skip-test=FILE-6310
skip-test=HTTP-6640
skip-test=HTTP-6643
skip-test=LOGG-2154
skip-test=LOGG-2190
skip-test=NETW-3200
skip-test=FIRE-4513
skip-test=ACCT-9628
skip-test=TIME-3116
skip-test=TIME-3120
skip-test=AUTH-9229
skip-test=TIME-3124
skip-test=TIME-3128
skip-test=KRNL-6000
skip-test=TOOL-5002
skip-test=FINT-4350 # пока не работает aide
skip-test=CRYP-7902 # пока не разобрался с одним из сертификатов
skip-test=HRDN-7222 # не работает paru



sbctl sign -s /efi/EFI/Microsoft/Boot/bootmgfw.efi && \
sbctl sign -s /efi/EFI/Microsoft/Boot/bootmgr.efi && \
sbctl sign -s /efi/EFI/Microsoft/Boot/memtest.efi
