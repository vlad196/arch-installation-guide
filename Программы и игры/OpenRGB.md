**Установка:**
```bash
paru -S --needed openrgb i2c-tools
```
**Включаем i2c-dev, создаём группу, если её нет и входим туда:**
```bash
sudo modprobe i2c-dev || \
sudo groupadd --system i2c || \
sudo usermod $USER -aG i2c
```
**Создаём правило для запуска:**
```bash
sudo bash -c 'echo "i2c-dev" >> /etc/modules-load.d/i2c.conf'
```
