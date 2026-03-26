# Пара 1 - Linux-основы контейниризации

## Блок 1 - Namespaces
`lsns` - список namespace-ов системы
<img width="667" height="457" alt="Снимок экрана от 2026-03-26 16-59-52" src="https://github.com/user-attachments/assets/8300ec79-dc99-4428-8dd5-8fe320cdc9f7" />


`echo $$` - внутри нового PID namespace(должно быть 1 или маленькое число)
<img width="667" height="45" alt="Снимок экрана от 2026-03-26 17-01-15" src="https://github.com/user-attachments/assets/130c186e-8ffd-4316-91f0-cfefd62b1b65" />


`ip link` в новом NET namespace(только Io)
<img width="667" height="62" alt="Снимок экрана от 2026-03-26 17-02-03" src="https://github.com/user-attachments/assets/0da8183a-8565-46c6-839a-5eec7c4663ab" />


**Контрольный вопрос:** Почему после `exit` процессы хоста остались нетронутыми?

**Ответ:** После `exit` процессы хоста остались нетронутыми, потому-что `unshare` создаёт изолированные namespace только для дочерних процессов — изменения не затрагивают родительскую среду и существующие процессы системы.

## Блок 2 - cgroups
`cat /sys/fs/cgroup/mytest/cpu.max` - ваш лимит

**Контрольный вопрос:** Что произойдёт если лимит памяти превысить? (OOM-killer)

**Ответ:** ядро Linux автоматически завершит один или несколько процессов, чтобы освободить память и стабилизировать систему. Приоритет на завершение получают процессы с высоким oom_score (например, потребляющие много памяти).

## Блок 3 - chroot
`ls /` внутри chroot

P.s: Столкнулся с проблемой, что в chroot окружении нельзя выполнить ls(ругается на отсутствие библиотеки `$\text{libselinux.so.1}$`), поэтому использовал следующие команды
```bash
# Копируем зависимости для bash
for dep in $(ldd /bin/bash | awk '/=>/ {print $3}'); do
    cp --parents "$dep" /tmp/myroot/
done

# Копируем зависимости для ls
for dep in $(ldd /bin/ls | awk '/=>/ {print $3}'); do
    cp --parents "$dep" /tmp/myroot/
done

# Явно копируем динамический линкер (если ещё не скопирован)
cp /lib64/ld-linux-x86-64.so.2 /tmp/myroot/lib64/
```
