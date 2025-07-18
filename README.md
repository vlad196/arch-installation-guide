# Установка и настройка Arch Linux для Surface-Book:
### Содержание:
- [Устанавливаем Arch Linux](Установка%20Achlinux.md)
- [Настраиваем драйвера](/Драйвера)
- [Настраиваем загрузчики](/Загрузчики)
- [Устанавливаем программы и игры](/Программы%20и%20игры)
- [Настраиваем среды рабочего стола](/Среды%20рабочего%20стола)
- [Работаем с ядрами](Ядра)
- [Дополнительные заметки](/Дополнительные%20заметки)

## Особенности:
- Описана настройка pacman.conf.
- Описана использование репозитория проекта [linux-surface](https://github.com/linux-surface/linux-surface)
- Описана установка компонентов, драйверов и ядра проекта [linux-surface](https://github.com/linux-surface/linux-surface)
- Описана установка двух ядер: одно основное и одно запасное с базовыми настройками. #TODO
- Описана разметка корневого раздела и swap на зашифрованные разделы LUKS.
- Описана настройка UKI.
- Описана настройка и установка драйверов Nvidia или Nouveau на выбор. #TODO
- Описана настройка и включение Secure Boot.
- Описано встраивание ключей расшифровки LUKS в TPM модуль.
- Описана настройка PLYMOUTH.
- Описаны базовые настройки IPTABLES.
- Описана настройка Trim.
- Описаны такие компоненты, как AppArmor, UsbGuard, Maldet, Timeshift, ClamAv и др.

## Моя система:
#### Видеокарта: [NVIDIA GTX 940m с GDDR5](https://www.notebookcheck-ru.com/NVIDIA-Maxwell-GPU-940M-GDDR5.413890.0.html), архитектура Maxwell
#### Процессор: Intell i5-6300U архитектура Skylake
#### Разметка дисков:
```mermaid
flowchart LR
  subgraph NVME0N1
    direction LR
    subgraph NVME0N1P1
        direction TB
        fat32
    end
    subgraph NVME0N1P2
        direction TB
        swapfs
    end
    subgraph NVME0N1P3
        direction TB
        f2fs
    end
  end
  subgraph Linux Catalogue Structure
    direction LR
        /
        /efi
        end
    end
  end
end
fat32 -- 2Gb--> /efi
fat32 -- 2Gb--> boot
f2fs -- 499Gb--> /
swapfs -- 11Gb--> swap
```
