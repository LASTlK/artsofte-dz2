# Отчёт по домашней работе

## Проблема 1. Сервис не запускается, ошибка 203/EXEC

### Что было обнаружено

При первоначальной проверке состояния сервиса командой `systemctl status` сервис находился в состоянии циклического перезапуска с ошибкой выполнения исполняемого файла (`status=203/EXEC`):

```text
Active: activating (auto-restart) (Result: exit-code)
Process: 7629 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)
Main PID: 7629 (code=exited, status=203/EXEC)
```

Код ошибки systemd `203/EXEC` указывает на то, что процессу не удалось выполнить бинарный файл или скрипт, указанный в директиве `ExecStart`.

### Как проводилась диагностика

1. Проверен точный путь `ExecStart`, используемый systemd:
   ```bash
   systemctl cat homework-app.service | grep ExecStart
   ```
   Вывод: `ExecStart=/opt/linux-devops-homework/app.py`. Это значение systemd считывает из `/etc/systemd/system/homework-app.service`.

2. Проверено фактическое содержимое каталога приложения `/opt/linux-devops-homework/`:
   ```bash
   ls -la /opt/linux-devops-homework/
   ```
   В каталоге находился только файл `server.py` с правами `-rwxr-xr-x` (владелец `root`). Файл `app.py` полностью отсутствовал.

3. Проанализированы исходные файлы репозитория и скрипты стенда:
   - В репозитории в `systemd/homework-app.service` было прописано `ExecStart=/opt/linux-devops-homework/app.py`.
   - В `scripts/setup.sh` установка выполнялась командой:
     `install -m 0755 "${REPO_DIR}/app/server.py" "${APP_DIR}/server.py"`
     (файл копируется как `server.py` в корень `/opt/linux-devops-homework/`).
   - В `scripts/check.sh` проверяется наличие в `ExecStart` точной строки `/opt/linux-devops-homework/app/server.py` (с подкаталогом `app/`).

### В чём была причина

Директива `ExecStart` в unit-файле ссылалась на несуществующий файл `app.py`. Кроме того, тестовый скрипт `check.sh` ожидает путь `/opt/linux-devops-homework/app/server.py`, в то время как установочный скрипт `setup.sh` копирует файл в плоскую структуру `/opt/linux-devops-homework/server.py`.

### Что было изменено

1. В репозитории в файле `systemd/homework-app.service` исправлен путь запуска:
   ```ini
   ExecStart=/opt/linux-devops-homework/app/server.py
   ```

2. В файловой системе целевого хоста создан подкаталог `app/` и перемещён файл приложения:
   ```bash
   sudo mkdir -p /opt/linux-devops-homework/app
   sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
   ```

3. Обновлён системный unit-файл и перезагружена конфигурация systemd:
   ```bash
   sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
   sudo systemctl daemon-reload
   sudo systemctl restart homework-app.service
   ```

### Почему было выбрано это решение

По условиям задания скрипты `scripts/setup.sh`, `scripts/check.sh` и `app/server.py` модифицировать запрещено. Единственным корректным решением является обновление пути в `systemd/homework-app.service` до `/opt/linux-devops-homework/app/server.py` и адаптация файловой структуры в `/opt/linux-devops-homework/app/`.
Команда `systemctl daemon-reload` необходима, так как systemd кэширует конфигурации unit-файлов в памяти.

### Как проверялся результат

1. Проверено актуальное значение `ExecStart` в systemd:
   ```bash
   systemctl cat homework-app.service | grep ExecStart
   ```
   Получен вывод: `ExecStart=/opt/linux-devops-homework/app/server.py`.
2. Запущен проверочный скрипт:
   ```bash
   sudo ./scripts/check.sh
   ```
   Проверка `[PASS] systemd ExecStart points to the application` успешно пройдена. Ошибка `203/EXEC` устранена, статус сменился на `1/FAILURE` (переход к следующей проблеме).

---

## Проблема 2. Ошибка прав доступа к директории состояния (`status=1/FAILURE`, `Permission denied`)

### Что было обнаружено

После исправления `ExecStart` сервис перестал падать на стадии exec, но стал аварийно завершаться при выполнении самого Python-скрипта (`status=1/FAILURE`):

```text
Process: 11092 ExecStart=/opt/linux-devops-homework/app/server.py (code=exited, status=1/FAILURE)
```

### Как проводилась диагностика

1. Просмотрен системный журнал сервиса:
   ```bash
   sudo journalctl -u homework-app.service -b --no-pager -l -n 50
   ```
   В журнале обнаружен Python traceback:
   ```text
   File "/opt/linux-devops-homework/app/server.py", line 18, in <module>
     with (STATE_DIR / 'startup.log').open('a', encoding='utf-8') as f:
   PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
   ```

2. Проверены права и владелец каталога состояния:
   ```bash
   ls -ld /var/lib/linux-devops-homework
   stat /var/lib/linux-devops-homework
   ```
   Каталог имел владельца `root:root` и права `0700` (`drwx------`).

3. Проверена конфигурация пользователя сервиса в unit-файле:
   В секции `[Service]` указаны `User=homework` и `Group=homework`. Процесс, запущенный от непривилегированного пользователя `homework`, не имел прав на создание и запись файлов в каталоге `/var/lib/linux-devops-homework`, принадлежащем `root` с правами `0700`.

### В чём была причина

Каталог данных `/var/lib/linux-devops-homework` был создан от пользователя `root` с правами `0700`. Приложение, запускаемое от системного пользователя `homework`, не могло создать файл `startup.log` и падало с `PermissionError: [Errno 13] Permission denied`.

### Что было изменено

Изменён владелец и права доступа каталога состояния согласно тз:

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

В соответствии с пунктом 3 требований раздела «Конечное состояние» README, каталог `/var/lib/linux-devops-homework` должен принадлежать `homework:homework` и иметь маску прав `0750` (`rwxr-x---`).
Это предоставляет пользователю `homework` полные права (чтение, запись, исполнение), группе `homework` — чтение и переход, и полностью запрещает доступ для остальных пользователей системы.

### Как проверялся результат

1. Проверены атрибуты каталога:
   ```bash
   ls -ld /var/lib/linux-devops-homework
   ```
   Вывод: `drwxr-x--- 2 homework homework 4096 ... /var/lib/linux-devops-homework`.
2. Проверен статус сервиса:
   ```bash
   systemctl status homework-app.service
   ```
   Сервис перешёл в статус `active (running)`.
3. Запущен проверочный скрипт:
   ```bash
   sudo ./scripts/check.sh
   ```
   Получены успешные статусы:
   - `[PASS] state directory ownership and permissions are correct`
   - `[PASS] service is active`
   - `[PASS] running process has the expected UID`

---

## Проблема 3. Приложение слушает только локальный интерфейс (loopback)

### Что было обнаружено

При запуске `sudo ./scripts/check.sh` была получена ошибка:
```text
[FAIL] TCP/8080 is reachable only through loopback
```
Сервис работал, но принимал подключения только с локального адреса `127.0.0.1:8080`.

### Как проводилась диагностика

1. Проверены открытые сетевые сокеты в системе:
   ```bash
   ss -lntp | grep 8080
   ```
   Вывод: `LISTEN 0 128 127.0.0.1:8080 users:(("server.py",pid=...,fd=3))`.
   Сокет привязан исключительно к интерфейсу `127.0.0.1`.

2. Проверен конфигурационный файл сервиса:
   ```bash
   cat /etc/linux-devops-homework/app.conf
   ```
   Содержимое:
   ```ini
   [server]
   host = 127.0.0.1
   port = 8080
   ```
   В файле репозитория `config/app.conf` также было указано `host = 127.0.0.1`.

### В чём была причина

Параметр `host = 127.0.0.1` указывает приложению слушать только loopback-интерфейс (локальную петлю). Пункт 6 требований раздела «Конечное состояние» требует слушать TCP-порт 8080 на всех IPv4-интерфейсах (`0.0.0.0:8080`).

### Что было изменено

1. В репозитории в файле `config/app.conf` изменён адрес привязки:
   ```ini
   [server]
   host = 0.0.0.0
   port = 8080
   ```

2. Обновлён конфигурационный файл в системе и перезапущен сервис:
   ```bash
   sudo cp config/app.conf /etc/linux-devops-homework/app.conf
   sudo systemctl restart homework-app.service
   ```

### Почему было выбрано это решение

Значение `0.0.0.0` указывает серверу принимать входящие соединения на указанном порту со всех доступных IPv4 сетевых интерфейсов хоста.
Конфигурационный файл `config/app.conf` разрешён к изменению правилами задания, а копирование в `/etc/linux-devops-homework/app.conf` синхронизирует системную конфигурацию с репозиторием.

### Как проверялся результат

1. Проверено состояние прослушивания сокетов:
   ```bash
   ss -lntp | grep 8080
   ```
   Вывод: `LISTEN 0 128 0.0.0.0:8080 users:(("server.py",pid=...,fd=3))`.
2. Проверен HTTP-эндпоинт приложения:
   ```bash
   curl -v http://127.0.0.1:8080/health
   ```
   Получен ответ `HTTP/1.1 200 OK` с телом `{"status": "ok"}`.
3. Запущен проверочный скрипт:
   ```bash
   sudo ./scripts/check.sh
   ```
   Все проверки успешно пройдены (`10 passed, 0 failed`).

---

## Дополнительные наблюдения

### 1. Исследование процесса через `/proc` и утилиты управления процессами

Для подтверждения соответствия запущенного процесса критериям безопасности и изоляции выполнен сбор диагностических данных:

- **Идентификатор процесса (PID) и родительский процесс (PPID):**
  ```bash
  MAIN_PID=$(systemctl show homework-app.service -p MainPID --value)
  ps -o pid,ppid,user,group,cmd -p "$MAIN_PID"
  ```
  Сервис имеет `PPID = 1` (управляется напрямую `systemd`), пользователь процесса — `homework`, группа — `homework`.

- **Проверка идентификаторов пользователя и группы через `/proc/<PID>/status`:**
  ```bash
  cat "/proc/$MAIN_PID/status" | grep -E '^(Uid|Gid):'
  ```
  Вывод:
  `Uid:  1001  1001  1001  1001` (Real, Effective, Saved, Filesystem UID соответствуют пользователю `homework`).
  `Gid:  1001  1001  1001  1001` (соответствуют группе `homework`).

- **Проверка аргументов запуска через `/proc/<PID>/cmdline`:**
  ```bash
  tr '\0' ' ' < "/proc/$MAIN_PID/cmdline"
  ```
  Вывод: `/usr/bin/python3 /opt/linux-devops-homework/app/server.py`. Подтверждает исполнение целевого скрипта через установленный в системе Python-интерпретатор.

- **Проверка файловых дескрипторов процесса через `/proc/<PID>/fd/`:**
  ```bash
  sudo ls -l "/proc/$MAIN_PID/fd/"
  ```
  Дескрипторы указывают на открытые потоки ввода/вывода (перенаправленные systemd в journald), файл журнала состояния `/var/lib/linux-devops-homework/startup.log` и listening TCP сокет.

### 2. Маршрутизация трафика (`ip route get 1.1.1.1`)

Для ответа на вопрос о выборе маршрута до публичного адреса `1.1.1.1`:
```bash
ip route get 1.1.1.1
```
Команда обращается к таблице маршрутизации ядра Linux и определяет маршрут по умолчанию (`default via <gateway_ip> dev <interface> src <source_ip>`). Для вычисления маршрута ядру не требуется фактическое подключение к интернету или отправка пакетов.

---

## Итоговый ход диагностики

**Шаг 1:** Первичный запуск `systemctl status homework-app.service` выявил ошибку `203/EXEC`.

**Шаг 2:** Анализ `systemctl cat` и содержимого `/opt/linux-devops-homework/` показал отсутствие `app.py` и несовпадение путей с требованиями `check.sh`.

**Шаг 3:** Исправлен `ExecStart` в `systemd/homework-app.service` на `/opt/linux-devops-homework/app/server.py`, создан каталог `/opt/linux-devops-homework/app/`, перемещён файл приложения, выполнен `systemctl daemon-reload` и перезапуск сервиса.

**Шаг 4:** Сервис перешёл в ошибку `1/FAILURE`. Через `journalctl -u homework-app.service` обнаружен `Permission denied` при записи в `/var/lib/linux-devops-homework/startup.log`.

**Шаг 5:** Анализ `ls -ld` выявил владельца каталога `root:root` и права `0700`. Каталогу назначены владелец `homework:homework` и права `0750`.

**Шаг 6:** Сервис успешно запустился в статусе `active`, однако `check.sh` указал на привязку только к `127.0.0.1:8080`.

**Шаг 7:** В конфигурации `config/app.conf` и `/etc/linux-devops-homework/app.conf` параметр `host` изменён на `0.0.0.0`, сервис перезапущен.

**Шаг 8:** Финальный запуск `sudo ./scripts/check.sh` подтвердил успешное прохождение всех 10 проверок.

---

## Сводка системных действий (не отражающихся в Git)

При повторном развёртывании стенда скриптом `scripts/setup.sh` необходимо выполнить следующие системные команды для восстановления рабочего состояния:

```bash
# 1. Приведение структуры каталога приложения к ожидаемой check.sh
sudo mkdir -p /opt/linux-devops-homework/app
sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py

# 2. Установка исправленного unit-файла и перезагрузка systemd
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service

# 3. Настройка владельца и прав каталога данных
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework

# 4. Установка исправленной конфигурации и перезапуск сервиса
sudo cp config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```

---

## Итоговая проверка

Вывод успешного запуска проверочного скрипта `sudo ./scripts/check.sh`:

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
