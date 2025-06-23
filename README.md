# Установка и настройка Arch Linux:
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
- Описана установка двух ядер: одно основное и одно запасное с базовыми настройками.
- Описана разметка корневого раздела и swap на зашифрованные разделы LUKS.
- Описана настройка UKI.
- Описана настройка и установка драйверов Nvidia или Nouveau на выбор.
- Описана настройка и включение Secure Boot.
- Описано встраивание ключей расшифровки LUKS в TPM модуль.
- Описана настройка PLYMOUTH.
- Описаны базовые настройки IPTABLES.
- Описана настройка Trim.
- Описаны такие компоненты, как AppArmor, UsbGuard, Maldet, Timeshift, ClamAv и др.

## Моя система:
#### Видеокарта: NVIDIA архитектура Turing
#### Процессор: AMD архитектура Zen2
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
        subgraph /mnt
            /mnt/sdb
        end
    end
    subgraph sdb
        direction TB
        btrfs
    end
    subgraph sda
        direction TB
        subgraph ntfs
    end
  end
  subgraph Windows
    direction LR
      boot
      C:/
  end
  

fat32 -- 2Gb--> /efi
fat32 -- 2Gb--> boot
f2fs -- 935Gb--> /
swapfs -- 16Gb--> swap
btrfs -- 2Tb--> /mnt/sdb
ntfs -- 256Gb--> C:/
```
