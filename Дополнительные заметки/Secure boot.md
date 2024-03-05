#Статья 
___
# Secure Boot
Это инструмент UEFI, позволяющая использовать белый список для запуска подписанных файлов при загрузке (Обычно EFI файлы)
## Установка
На странице Secure boot очень хорошо раскрыта схема установки (помимо нижних этапов есть очень много других объяснений про предзагрузчики, про разные хуки, про лист разрешённых модулей и т.п.) 
[Unified Extensible Firmware Interface/Secure Boot - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot) 
На странице по Secure boot очень много написано вариантов, но основной механизм подписи такой: 
0.  Удаляем старые ключи в uefi 
1.  Создаём ключи и сертификаты для системы 
2.  Соединяем их с ключами и сертификатами Майкрософт, т.к. у нас есть Windows и, возможно некоторые устройства на материнке требуют (а может просто добавляем в защищённый список, который создаётся на этапе 1) 
3.  Делаем enroll в систему  
4.  Подписываем все исполняемые файлы в загрузчике 
5.  Озадачиваемся о самостоятельной самоподписи после каждого обновления ядра, загрузчика и т.п. 

## SBCTL
Есть очень простой механизм установки, благодаря sbctl:
```bash
sudo sbctl create-keys 

sudo sbctl enroll-keys -m # -m означает, что ключи будут уже вместе  
# с Майкрософт сертификатами 

sudo sbctl sign -s /boot/vmlinuz-linux 
sudo sbctl sign -s /boot/vmlinuz-linux-lqx 
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI 
sudo sbctl sign -s /boot/EFI/Linux/fallback.efi 
sudo sbctl sign -s /boot/EFI/Linux/fallback-lqx.efi 
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi 
sudo sbctl sign -s /boot/EFI/BOOT/bootx64.efi 
sudo sbctl sign -s /boot/EFI/BOOT/Microsoft/bootmgfw.efi 
sudo sbctl sign -s /boot/EFI/BOOT/Microsoft/bootmgr.efi 
sudo sbctl sign -s /boot/EFI/BOOT/Microsoft/memtest.efi 

# Проверяем всё ли необходимое подписано 
sudo sbctl list-files 

#Проверяем как как работает подписывание hook 
pacman -S linux # переустанавливаем 

# и снова проверяем 
sudo sbctl list-files 
# Если signed, значит всё хорошо
```
И всё. По идее уже сам systemd-boot отлично справляется и запускается

#### А если нужно опять подписать уже сохранённые в таблицах файлы, то:
```bash
sudo sbctl sign-all
```

#### Бэкап прежней загрузки:
```bash
Yay –S efitools 


sudo efi-readvar -v PK -o old_PK.esl  

sudo efi-readvar -v KEK -o old_KEK.esl  

sudo efi-readvar -v db -o old_db.esl  

sudo efi-readvar -v dbx -o old_dbx.esl
```
___
Ссылки:
[[Secure boot]] [[sbctl]] [[boot]] [[EFI]] [[UEFI]]