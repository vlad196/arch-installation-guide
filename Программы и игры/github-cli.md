#Github-cli

## Установка:

```bash
sudo pacman -S github-cli
```
## Авторизация:

```bash
gh auth login
```
В моём случае настройки подходят такие:
- GitHub.com
- SSH
- no generate SSH (Если уже есть)
- Login with a web browser

## Смотри наши проекты:

```bash
gh repo list
```

## Копируем уже существующий проект:
Так как у меня был плохой опыт с venv каталогами на разных системах при синхронизации через yandex-disk, лучше синхронизировать через GitHub, и синхронизровать только нужные файлы.

bin это каталог для бинарных файлов. Для проектов, лучше испрльзовать projects или, лучше, наверное repositories

```bash
mkdir -p ~/repositories
```

```bash
gh repo clone owner/repo
```
