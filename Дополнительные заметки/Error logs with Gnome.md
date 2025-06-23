Ошибка:
```bash
gsd-sharing[981]: Failed to StopUnit service: GDBus.Error:org.freedesktop.DBus.Error.Spawn.ChildExited: Process org.freedesktop.systemd1 exited with status>
сен 20 18:15:41 arch gsd-sharing[981]: Failed to StopUnit service: GDBus.Error:org.freedesktop.DBus.Error.Spawn.ChildExited: Process org.freedesktop.systemd1 exited with status>
сен 20 18:15:47 arch gdm-password][1570]: gkr-pam: unable to locate daemon control file
```
>Ответ:
>Это набор сообщений, которые принадлежат друг другу.
>
>Он не находит управляющий файл демона, поэтому кэширует пароль и успешно открывает связку ключей после правильной инициализации сеанса.
>
>Читайте, нет проблем и ошибок, и это имеет информационную ценность. Это может быть актуально, если демон не запускается во время инициализации сеанса, так как здесь это не так, его можно спокойно игнорировать.
>
>Основная причина, вероятно, заключается в том, что gkr-pam запускается до того, как система инициализирует оставшуюся часть сеанса и запускает соответствующие пользовательские службы. Поэтому демон gnome-keyring, предназначенный для разблокировки, еще не запускается при первой проверке.
>https://bbs.archlinux.org/viewtopic.php?id=261156

