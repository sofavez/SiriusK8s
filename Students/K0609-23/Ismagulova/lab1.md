# LAB1
---
##block1
Сначала просматриваются все пространства имен, типы: cgroup, ipc, mnt (mount), net, pid, user, uts, а также специфичные для дочерних процессов pid_for_children и time_for_children. Команда lsnc там выведет их количество (их там много).
<img width="1230" height="664" alt="изображение" src="https://github.com/user-attachments/assets/0d2553dc-21f2-4273-ae97-4f00ae22b6b7" />

Далее создается новый PID namespace, внутри которого отсчет идентификаторов процессов (PID) начинается заново и туда еще заходим и проверяем что мы PID 1.
<img width="781" height="315" alt="изображение" src="https://github.com/user-attachments/assets/b832ae9d-aba8-4620-8893-432d1ea02aa3" />

Дальше создается пустой стек, в котором можно увидеть один сетевой интерфейс, который по умолчанию будет выключен
<img width="888" height="168" alt="изображение" src="https://github.com/user-attachments/assets/a6ec053c-e45e-433a-8d94-4f90ba48c65b" />
