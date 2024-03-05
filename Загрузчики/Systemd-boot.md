
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
> Ранее был grub, но в итоге лично мне не понравился по нескольким причинам:
> а) графический интерфейс хоть и кастомизированный, но тормозящий
> б) нет поддержки именно UKI, что в systemd-boot это уже отдельная от EFIshell сущность
> в) grub слишком громоздкий и его функционал для моих хотелок слишком большой
> Минус ухода от grub в том, что я лишаюсь возможности красиво назвать UKI и не могу по своему распределить меню выбора
