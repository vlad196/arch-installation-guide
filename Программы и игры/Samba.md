### Утановка Samba
```bash
sudo pacman -S samba
```

### Настройка

Глобальная настройка и медиа папка:
```bash
sudo bash -c 'cat << _EOF_ > /etc/samba/smb.conf
[global]
# Рабочая группа Windows
workgroup = WORKGROUP
server string = Samba
server min protocol = SMB3_11
server max protocol = SMB3_11
server smb encrypt = desired
security = user
# Windows против гостевых режимов, поэтому отключаем
map to guest = never
fruit:copyfile = yes
create mask = 0664
hide dot files = yes
# Для UserShares
usershare path = /var/lib/samba/usershares
usershare max shares = 100
usershare allow guests = yes
usershare owner only = yes

[media]
comment = Public Folder
#Тут пишем путь до папки для шары
path = /mnt/sdb/Видео/
browseable = Yes
guest ok = no
public = no
writeable = Yes
read only = no
guest ok = yes
create mask = 0666
directory mask = 0775
_EOF_`

Настройка UFW:
```bash
sudo ufw allow CIFS
```
### Samba пользователи

Samba использует Linux пользователей, но имеет собственную базу паролей.
Чтобы добавить пароль в Samba пользователю:
```bash
sudo smbpasswd -a <пользователь>
```
>[!INFO]
>Eсли вы хотите разрешить новому пользователю только доступ к Samba-ресурсам и запретить полноценный вход в систему, можно ограничить возможности входа:
>- отключить командную оболочку - `usermod --shell /usr/bin/nologin --lock _пользователь_samba_`
>- отключить вход по ssh- измените опцию `AllowUsers` в файле `/etc/ssh/sshd_config`

Просмотр списка пользователей:
```bash
sudo pdbedit -L -v
```
### Usershares

Это возможность, позволяющая обычным пользователям добавлять, изменять и удалять собственные ресурсы общего доступа.

1. Создайте каталог, в котором будут храниться описания пользовательских общих ресурсов:
``` bash
sudo mkdir /var/lib/samba/usershares
```
2. Создайте [группу](https://wiki.archlinux.org/title/Users_and_groups_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)#Управление_группами "Users and groups (Русский)") для пользователей, которые смогут создавать общие ресурсы:

```bash
sudo groupadd -r sambashare
```

3. Измените владельца каталога на `root`, а группу на `sambashare`:

```bash
sudo chown root:sambashare /var/lib/samba/usershares
```

4. Измените разрешения каталога `usershares`, чтобы только пользователи из группы `sambashare` могли создавать файлы. Эта команда также устанавливает [sticky bit](https://en.wikipedia.org/wiki/ru:Sticky_bit "wikipedia:ru:Sticky bit"), благодаря которому пользователи не смогут удалять чужие общие ресурсы:

```bash
sudo chmod 1770 /var/lib/samba/usershares
```

Задайте эти переменные в конфигурационном файле (Но мы ранее это уже сдеали):
```bash
[global]
usershare path = /var/lib/samba/usershares
usershare max shares = 100
usershare allow guests = yes
usershare owner only = yes
```

Добавьте вашего пользователя в группу sambashare. Замените `ваше_имя_пользователя` на имя вашего linux-пользователя:

```bash
sudo gpasswd sambashare -a <ваше_имя_пользователя>
```

[Перезапустите](https://wiki.archlinux.org/title/%D0%9F%D0%B5%D1%80%D0%B5%D0%B7%D0%B0%D0%BF%D1%83%D1%81%D1%82%D0%B8%D1%82%D0%B5 "Перезапустите") службы `smb.service`:
```bash
sudo systemctl restart smb
```
Завершите сеанс и войдите снова, чтобы применилось добавление новой группы к вашему пользователю.

Если вы хотите предоставить общий доступ к файлам, находящимся в вашем домашнем каталоге, не забудьте задать доступ как минимум на чтение другим пользователям (`chmod a+rX`).
