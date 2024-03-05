**Установка clamav и clamtk:**
```bash
paru -S clamav clamtk
```

**Закомментируем заблокированное зеркало и добавляем новое:**
```bash
sudo bash -c "sed '/DatabaseMirror database.clamav.net/s/^/#/' -i /etc/clamav/freshclam.conf"  &&\
sudo bash -c "sed '/DatabaseMirror /a DatabaseMirror https://pivotal-clamav-mirror.s3.amazonaws.com' -i /etc/clamav/freshclam.conf"
```

**Включаем демон clamav:**
```bash
sudo systemctl enable --now clamav-daemon.service
```
>[!Info]
>Информация взята отсюда:
>https://interface31.ru/tech_it/2022/06/nastraivaem-antivirusnuyu-zashhitu-v-real-nom-vremeni-na-osnove.html