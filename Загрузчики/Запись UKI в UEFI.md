
> [!INFO]
> Т.к. мы используем UKI, можно сделать загрузку напрямую без загрузчика из UEFI
> Есть минусы:
> - Не будут работать такие вещи как fwupd
> - Не будет никаких дополнительных вариантов загрузки
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader 'EFI\Linux\arch-linux-lqx.efi' --unicode
```
