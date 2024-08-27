**Установка:**
```bash
paru -S usbguard
```
**Разрешаем все подключенные устройства:**
```bash
sudo bash -c "usbguard generate-policy > /etc/usbguard/rules.conf"
```
**Включаем usbguard, создаём группу и входим туда:**
```bash
sudo groupadd --system usbguard || \
sudo usermod $USER -aG usbguard
```
Добавляем
**Включаем работу IPC группы usbguard:**
```bash
sudo bash -c "sed '/IPCAllowedGroups=/s/\$/usbguard/' -i /etc/usbguard/usbguard-daemon.conf"
```
 >[!Note]
>В вики arch описана работа с wheel группой, но для единобразия, я сделал отдельную группу по подобию отдельной группы audit.

**Включаем dbus версию юнита (а иначе всякие gui приложения работать не будут):**
```bash
sudo systemctl enable --now usbguard-dbus.service
```
>[!Note]
>Если нужен чисто CLI клиент, то достаточно usbguard.service

#### Usbguard qt апплет:
**Устанавливаем usbguard апплет:**
```bash
paru -S usbguard-qt
```
