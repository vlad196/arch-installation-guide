### Генерация новых правил для приложения:
1. Отключаем лимит rate limit на ядре:
```bash
sudo sysctl -w kernel.printk_ratelimit=0
```
2. Запускаем aa-genprof для программы:
```bash
sudo aa-genprof <program>
```
3. Работаем в приложении. Если уже есть информация, где AppArmor может заблокировать, повторяем именно это.
4. В консоли с запущенным `aa-genprof` сканируем логи (S), а потом завершаем (F).
5. Мы запускаем приложение в режиме `aa-complain`:
```bash
sudo aa-complain <program>
```
6. Для детальной настройки запускаем `aa-logprof` и решаем, что делать:
```bash
sudo aa-logprof
```
TODO: добавим описания опций.
7. Запустим приложение в режиме `aa-enforce`:
```bash
sudo aa-enforce <program>
```

>[!Note]
>Подробную информацию мы можем найти здесь: https://gitlab.com/apparmor/apparmor/-/wikis/Profiling_with_tools


>[!Info]
>Wiki по AppArmor: https://gitlab.com/apparmor/apparmor/-/wikis/home
