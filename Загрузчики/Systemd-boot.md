
**Установка Systemd-boot:**
```bash
 bootctl install
```
**Подписываем systemd-boot файл:**
```bash
sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi &&\
sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi 
```
#### Конфигурируем systemd-boot:
>[!Note]
>Не забыть про переменную $MAIN_KERNEL

**В папке loader создать и настроить:**
```bash
cat << _EOF_ > /efi/loader/loader.conf
default arch-$MAIN_KERNEL.efi
timeout 10
console-mode max
editor no
_EOF_
```
> [!INFO]
> Ранее был grub, но в итоге лично мне не понравился по нескольким причинам:\
> а) графический интерфейс хоть и кастомизированный, но тормозящий\
> б) не знает что такое UKI, относя это к обычным EFI приложениям. В systemd-boot это отдельная от EFIshell сущность\
> в) grub слишком громоздкий и его функционал для моих хотелок слишком большой
>
> Минус ухода от grub в том, что я лишаюсь возможности красиво назвать UKI и не могу по своему распределить меню выбора

### Дополнительные настройки:

**Добавление других загрузчиков с других дисков:**

Добавляем PARTUUID раздела с другим загрузчиком и названием в переменные:
```bash
export PART_ANOTHER_BOOT_1=$(lsblk -dno UUID /dev/sdc1)
export Name_ANOTHER_OS_1="Alt linux"cat << _EOF_ > /efi/loader/entries/Alt-linux.conf
title  Alt-linux
efi     /shellx64.efi
options -nointerrupt -noconsolein -noconsoleout Alt-linux.nsh
_EOF_
```
Создаём внешний скрипт для запуска с другого диска:
```bash
cat << _EOF_ > /efi/Alt-linux.nsh
$PART_ANOTHER_BOOT_1:#TODO дописать местоположение EFI grub
_EOF_
```
Добавляем новую точку входа:
```bash
cat << _EOF_ > /efi/loader/entries/Alt-linux.conf
title  Alt-linux
efi     /shellx64.efi
options -nointerrupt -noconsolein -noconsoleout Alt-linux.nsh
_EOF_
```
