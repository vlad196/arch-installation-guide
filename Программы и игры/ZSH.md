### ZSH
**Установка:**
```bash
paru -S zsh zsh-completions awesome-terminal-fonts
```
**Установка oh-my-zsh:**
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
**Установить тему для zsh:**
```bash
paru -S zsh-theme-powerlevel10k-git
```

```bash
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

```
zsh
```
**Сменяем дефолтный Shell на zsh:**
```bash
chsh -s /bin/zsh
```