# Установка aes67-daemon

Установка aes67-daemon производится из репозитория common-deb

К роли жестко привязана версия 1.6.5-1

Запуск роли: `ansible-playbook dante.yml -i inventory/hosts.yml`

## Описание
Эта Ansible-роль предназначена для установки и настройки сервиса aes67-daemon, виртуальной карты RAWENNA, удаления PulseAudio, а также настройки сетевого взаимодействия через `netplan` и `sysctl`.

Роль не удаляет PulseAudio с миниПК конференции, так как конференция без PulseAudio работает не корректно. Тем не менее в некоторых случаях после удаления PulseAudio с миниПК конференции, она работает корректно. 

## Обзор задач

### 1. Получение сетевых настроек
Задача `Get network settings` определяет IP-адреса для различных VLAN на основе IP-адреса хоста:
- `dante_ip` — IP-адрес в сети Dante.
- `vlan30_ip` — IP-адрес в VLAN 30.
- `vlan2_ip` — IP-адрес в VLAN 2.

### 2. Удаление PulseAudio
Если хост не принадлежит группе `conference`, выполняются следующие шаги:
- `Purge PulseAudio` — Удаление пакета `pulseaudio` вместе с зависимостями.
- `Remove PulseAudio trash` — Удаление остаточных конфигурационных файлов PulseAudio.

### 3. Установка aes67-daemon
- Устанавливает пакет `aes67-daemon` указанной версии.
- После установки выполняется перезагрузка хоста.

### 4. Настройка сети через Netplan
- Копируется template файл `netplan` в зависимости от группы хоста (`tap` или `gers`).
- После изменения конфигурации требуется перезагрузка хоста.

### 5. Копирование конфигурации aes67-daemon
- `Copy aes config` — Устанавливает конфигурационный файл `/etc/aes67-daemon/daemon.conf`.
- `Copy aes sound source` — Копирует файл `/var/lib/aes67-daemon/status.json`, если хост принадлежит группе `tap`. В файле содержится информация на основе которой aes67 автоматически создаст source для хостов, которые отдают звук

### 6. Настройка PulseAudio (для группы conference)
- В файле `/etc/pulse/daemon.conf` заменяется строка `; default-sample-rate = 44100` на `default-sample-rate = 48000`. Это нужно для корректной передачи звука на конференции.
- После изменения требуется перезагрузка хоста.

### 7. Конфигурация Systemd-сервисов
- `Make override directory` — Создает директорию `/etc/systemd/system/{{ service_name }}.service.d` для переопределения параметров systemd-сервиса.
- `Create the override configuration` — Копирует `override.conf` в созданную директорию и перезапускает сервис `Lyra`. Это позволит сделать службу Lyra зависимой от aes67.

### 8. Настройка параметров ядра (sysctl)
Задача `Set sysctl parameters` применяет настройки ядра для группы `tap` и `gers`, включая параметры производительности и сетевого взаимодействия.
- Применяются параметры реального времени для `tap`.
- Увеличиваются буферы и настройки TCP для `gers`.
- После применения требуется перезагрузка хоста.

## Обработчики (Handlers)
- `Restart aes67-daemon service` — Перезапускает сервис `aes67-daemon`.
- `Restart Lyra service` — Перезапускает сервис `Lyra`.
- `Reboot host` — Выполняет перезагрузку хоста (таймаут 180 секунд).

## Переменные
- `service_name` — Имя сервиса, который переопределяется в systemd.
- `aes_version` — Версия пакета `aes67-daemon`, которая устанавливается.
- `octet_1`, `octet_2` — Первые два октета IP-адреса (необходимо определить в inventory).
- `dante_ip` — IP-адрес в сети Dante.
- `vlan30_ip` — IP-адрес в VLAN 30.
- `vlan2_ip` — IP-адрес в VLAN 2.
