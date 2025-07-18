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
    %% Технические ID (NVME_P1, SDA_P1) здесь есть, но на картинке их не будет
    subgraph "Физические диски"
        direction TB
        subgraph "NVME0N1"
            NVME_P1["fat32 <br> EFI-раздел"]
            NVME_P2["swapfs"]
            NVME_P3["f2fs"]
        end

        subgraph "SDA"
            SDA_P1["ntfs"]
        end

        subgraph "SDB"
            SDB_P1["btrfs"]
        end
    end

    subgraph "Точки монтирования"
        direction TB
        subgraph "Linux"
            L_EFI["/efi"]
            L_ROOT["/"]
            L_SWAP["[SWAP]"]
            L_MNT_SDB["/mnt/sdb"]
        end

        subgraph "Windows"
            W_BOOT["boot"]
            W_C["C:/"]
        end
    end

    %% Связи используют невидимые технические ID
    NVME_P1 -- "2Gb" --> L_EFI
    NVME_P1 -- " " --> W_BOOT
    NVME_P2 -- "24Gb<br>(16Gb RAM + 8 Gb RTX 2070)" --> L_SWAP
    NVME_P3 -- "935Gb" --> L_ROOT
    SDA_P1  -- "256Gb" --> W_C
    SDB_P1  -- "2Tb" --> L_MNT_SDB
```
