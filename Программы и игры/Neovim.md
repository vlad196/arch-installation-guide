**Установка необходимых пакетов:**
```bash
paru -S --needed neovim npm ripgrep lazygit wl-clipboard tree-sitter
```
>[!Info]
>rigrep - инструмент, который ищет файлы по описанным патернам
>lazygit - утилита для работы с git
>wl-clipboard - утилита для передачи в буфер-обмена
>tree-sitter - утилита для работы с синтаксисом языков
### Astronvim
**Установка:**
```bash
git clone --depth 1 https://github.com/AstroNvim/template ~/.config/nvim
# remove template's git connection to set up your own later
rm -rf ~/.config/nvim/.git
nvim
```

#### Дальнейшая настройка:
```bash
nvim
```
**Установка LanguageServerProtocols:**
Смотреть сервера тут: https://microsoft.github.io/language-server-protocol/implementors/servers/
**Установка LSP серверов:**
```
:LspInstall pyright tsserver bashls
```
**Установка парсеров для языков:**
```
:TSInstall python javascript bash
```
**Установка дебаггеров для языков:**
```
:DapInstall python js bash
```
