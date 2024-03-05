**Установка:**
```bash
paru -S maldet
```
**Включение юнита maldet-update-signatures.timer:**
```bash
sudo systemctl enable --now maldet-update-signatures.timer
```
**Обновление maldet-update-signatures.timer:**
```bash
sudo systemctl start maldet-update-signatures.service
```