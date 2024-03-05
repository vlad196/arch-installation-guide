#Статья 
___
# Dotfiles
**Dotfiles** (файлы, папки с точкой) -- обычно, в Unix-like системах обозначает 
Файл или папку с конфигурациями. 
## Shell
**.bashrc .zshrc** и т.п., по сути, тоже являются файлами с конфигурациями для оболочек 
И называется файлы запуска оболочек 
Возьмём bash: 
 Есть 4 файла: **.bash_profile, .profile, .bash_login, .bashrc** 
В **.bash_profile** сидит ссылка на **.bashrc**. Это надо, т.к. в системе есть несколько разных типов экземпляров bash, где есть два основных -- интерактивные и не интерактивные 

Не интерактивные экземпляры оболочек не читают файлы запуска. 
Интерактивные используются для выполнения команд с терминала, и ещё к этому интерактивные оболочки клаcсифицируются как оболочки входа в систему и оболочки без входа в систему. 

Оболочка входа в систему появляется при первом входе в систему в терминале (tty, ssh,  Мы ещё настраивали первое сообщение ему) 
Bash запускает **/etc/profile/profile**, далее запускает один из первых попавшихся 
 **.bash_profile, .profile, .bash_login** 
Оболочки без входа в систему это любая интерактивная оболочка уже в появилась  
после входа в систему (консоль в гном, обычная смена оболочки внутри сессии) 
Bash запускает **/etc/bash.bashrc**, далее запускает **.bashrc**

Окружения бывают разными, мы сейчас затронули только пользовательское 
Вот какие есть ещё: 
-   Системное: **/etc/environment /etc/profile** 
-   На уровне пользователя: **.bash_profile, .profile, .bash_login, .bashrc** 
-   Дополнительно к пользовательскому: окружение приложения, X11, wayland 
*Важно: пользовательские переписывают системные*

#### Источники: 

1. [https://en.wikipedia.org/wiki/Hidden_file_and_hidden_directory#Unix_and_Unix-like_environments](https://en.wikipedia.org/wiki/Hidden_file_and_hidden_directory#Unix_and_Unix-like_environments) 

>In Unix-like operating systems, any file or folder that starts with a dot character (for example, /home/user/.config), commonly called a dot file or dotfile, is to be treated as hidden – that is, the ls command does not display them unless the -a or -A flags (ls -a or ls -A) are used.[5] In most command-line shells, wildcards will not match files whose names start with . unless the wildcard itself starts with an explicit . . 

>A convention arose of using dotfiles in the user's home directory to store per-user configuration or informational text. Early uses of this were the well-known dotfiles .profile, .login, and .cshrc, which are configuration files for the Bourne shell and C shell and shells compatible with them, and .plan and .project, both used by the finger and name commands.[6] 

>Many applications, from bash to desktop environments such as GNOME, now store their per-user configuration this way, but the Unix/Linux freedesktop.org XDG Base Directory Specification aims to migrate user config files from individual dotfiles in $HOME to non-hidden files in the hidden directory $HOME/.config.[7]  



2. [https://wiki.archlinux.org/title/Dotfiles#top-page](https://wiki.archlinux.org/title/Dotfiles#top-page)

>User-specific application configuration is traditionally stored in so called [dotfiles](https://en.wikipedia.org/wiki/dotfile) (files whose filename starts with a dot). It is common practice to track dotfiles with a [version control system](https://wiki.archlinux.org/title/Version_control_system) such as [Git](https://wiki.archlinux.org/title/Git) to keep track of changes and synchronize dotfiles across various hosts. There are various approaches to managing dotfiles (e.g. directly tracking dotfiles in the home directory v.s. storing them in a subdirectory and symlinking/copying/generating files with a [shell](https://wiki.archlinux.org/title/Shell) script or [a dedicated tool](https://wiki.archlinux.org/title/Dotfiles#Tools)). Apart from explaining how to manage dotfiles this article also contains [a list of dotfile repositories](https://wiki.archlinux.org/title/Dotfiles#User_repositories) from Arch Linux users.  

Ссылки: [[Envinronment]] [[Dorfile]] [[bash]]