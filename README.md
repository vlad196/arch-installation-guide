# Установка и настройки Arch Linux:
### Содержание:
- [Установка Arch Linux](Установка%20Achlinux.md)
- [Драйвера](/Драйвера)
- [Загрузчики](/Загрузчики)
- [Программы и игры](/Программы%20и%20игры)
- [Среды рабочего стола](/Среды%20рабочего%20стола)
- [Ядра](Ядра)
- [Дополнительные заметки](/Дополнительные%20заметки)

## Особенности:
- описана настройка pacman.conf
- описана установка 2 ядер. Одно основное, а второе запасное с базовыми настройками
- описана разметка корневого раздел и swap на зашифрованные разделы LUKS
- описана настройка UKI
- описана настройка и установка Nvidia или Nouveau драйверов на выбор 
- описана настройка и включение Secure boot
- описана встраивание ключей расшифровки LUKS в TPM модуль
- описана настройка PLYMOUTH
- описаны базовые настройки IPTABLES
- описана настройка Trim
- Описаны такие компоненты, как: AppArmor, UsbGuard, Maldet, Timeshift, ClamAv и др.

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
