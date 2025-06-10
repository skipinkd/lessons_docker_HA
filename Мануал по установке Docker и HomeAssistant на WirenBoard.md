# Установка Docker

## Подготовка к установке

> Обязательно расширить корневой раздел до 2гб!!!

Установите необходимые зависимости:
```bash
apt update && apt install ca-certificates curl gnupg lsb-release iptables
```

Добавьте репозиторий с пакетами docker:
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Добавьте GPG ключ для репозитория:
```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

**Если у вас релиз [wb-2304](https://wirenboard.com/wiki/Wb-2304 "Wb-2304") и новее**, надо выполнить команды (вставьте каждую строчку отдельно):
```bash
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

Это позволит Docker настраивать виртуальные сети и пробрасывать порты.

Если при установке Docker появится ошибка:
```bash
DEBU[2025-01-03T20:00:31.940760682+03:00] Cleaning up old mountid : done.              
failed to start daemon: Error initializing network controller: error creating default "bridge" network: Failed to Setup IP tables: Unable to enable NAT rule:  (iptables failed: iptables --wait -t nat -I POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE: iptables v1.8.7 (nf_tables): Chain 'MASQUERADE' does not exist
```

Создайте правило:
```bash
iptables -w10 -t nat -I POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```


## Предварительная настройка

Настройте симлинк для папки конфигурации:
```bash
mkdir /mnt/data/etc/docker && ln -s /mnt/data/etc/docker /etc/docker
```

Создайте папку для хранения образов:
```bash
mkdir /mnt/data/.docker
```

Укажите в файле настроек **daemon.json** созданную выше папку:
1. Откройте файл в редакторе:
	```bash
	nano /etc/docker/daemon.json
	```

2. Вставьте в него строки:
	```JSON
	{
		"data-root": "/mnt/data/.docker",
		"log-driver": "json-file",
		"log-opts": {
			"max-size": "10m",
			"max-file": "3"
		}
	}
	```

3. Сохраните и закройте файл:  ``Ctrl+S`` и ``Ctrl+X``.

## Установка

После того, как мы указали, где будут храниться контейнеры, устанавливаем сам docker:
```bash
apt update && apt install docker-ce docker-ce-cli containerd.io
```

Чтобы проверить, что всё работает — запустите контейнер ``hello-world``:
```bash
docker run hello-world
```

Если в консоли появилась надпись ``Hello from Docker!``, docker установлен и работает.

Иногда требуется перезагрузка командой:
```bash
reboot
```

затем повторить проверку.


# Установка HomeAssistant

## Установка
Создайте каталог под служебные файлы:
```bash
mkdir /mnt/data/.HA
```

Запустите образ homeassistant — docker автоматически загрузит его из интернет и запустит:
```bash
docker run -d --name homeassistant --privileged --restart=unless-stopped -e TZ=Europe/Moscow -v /run/dbus:/run/dbus:ro -v /mnt/data/.HA:/config --network=host ghcr.io/home-assistant/home-assistant:stable
```

## Полное копирование и перенос Home Assistant

Давайте разберем **полный процесс переноса контейнера вместе с volume**. 
Этот процесс включает:

1. Остановка контейнер
2. Создание резервной копии каталога ``/mnt/data/.HA``
3. Перенос архивв на новую систему
4. Распаковка данных на новой системе
5. Запуск контейнера на новой системе
6. Проверка работы

#### 1. Остановите контейнер

Перед началом переноса убедитесь, что контейнер остановлен, чтобы избежать повреждения данных.
```shell
docker stop homeassistant
```

#### 2. Создайте резервную копию папки ``/mnt/data/.HA``

Папка ``/mnt/data/.HA`` содержит все данные Home Assistant, включая:
* Конфигурационные файлы (``configuration.yaml``, ``automations.yaml``, ``scripts.yaml`` и т. д.).
* Аддоны (если они установлены через HACS или другие источники).
* Логи, базы данных и пользовательские данные.

Создайте архив этой папки:
```shell
tar -czvf HA_backup.tar.gz -C /mnt/data .HA
```

- `-C /mnt/data`: Указывает, что нужно сжать содержимое папки `.HA` из `/mnt/data`.
- `HA_backup.tar.gz`: Имя архива, который будет создан.

#### 3. Перенесите архив на новую систему

Скопируйте созданный архив на новую систему. Это можно сделать разными способами:

- Через USB-накопитель.
- Через SCP (если у вас есть доступ по SSH):
    ```shell
    scp homeassistant_backup.tar.gz user@new-server:/path/to/destination/
    ```
    

#### 4. Распакуйте данные на новой системе

На новой системе распакуйте архив в нужное место. Например, если вы хотите использовать ту же структуру (`/mnt/data/.HA`):
```shell
mkdir -p /mnt/data
tar -xzvf homeassistant_backup.tar.gz -C /mnt/data
```

Убедитесь, что права доступа к папке соответствуют требованиям Docker. Если необходимо, измените владельца папки:
```shell
chown -R $(id -u):$(id -g) /mnt/data/.HA
```

#### 5. Запустите контейнер на новой системе

Используйте ту же команду `docker run`, чтобы запустить контейнер. Например:
```shell
docker run -d \
  --name homeassistant \
  --privileged \
  --restart=unless-stopped \
  -e TZ=Europe/Moscow \
  -v /run/dbus:/run/dbus:ro \
  -v /mnt/data/.HA:/config \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable
```

#### 6. Проверьте работу

Откройте веб-интерфейс Home Assistant в браузере (`http://<IP-адрес>:8123`) и убедитесь, что:
- Все настройки сохранились.
- Аддоны работают корректно.
- Автоматизации и сценарии функционируют как раньше.

## Полное удаление Home Assistant

Чтобы полностью удалить Home Assistant с вашей системы, нужно выполнить несколько шагов. Это включает остановку и удаление контейнера, удаление связанных данных (например, папки `/mnt/data/.HA`), а также очистку Docker-образа, если это необходимо. Вот пошаговая инструкция:

#### 1. Остановите контейнер

Первым делом остановите работающий контейнер Home Assistant:
```shell
docker stop homeassistant
```

#### 2. Удалите контейнер

После остановки контейнера удалите его:
```shell
docker rm homeassistant
```

#### 3. Удалите данные Home Assistant

Home Assistant хранит все свои данные в папке `/mnt/data/.HA` (или той, которую вы указали в параметре `-v /mnt/data/.HA:/config`). Если вы хотите полностью удалить Home Assistant, удалите эту папку:

```shell
sudo rm -rf /mnt/data/.HA
```

**Важно**: Убедитесь, что вы действительно хотите удалить все данные, так как это действие необратимо.

#### 4. Удалите Docker-образ

Если вы больше не планируете использовать Home Assistant, можно удалить его Docker-образ. Сначала найдите ID образа:

```shell
docker images
```

Вы увидите что-то вроде:

```
REPOSITORY                              TAG       IMAGE ID       CREATED        SIZE
ghcr.io/home-assistant/home-assistant   stable    abc12345def6   2 weeks ago    1.2GB
```

Затем удалите образ:

```shell
docker rmi ghcr.io/home-assistant/home-assistant:stable
```

Или используйте `IMAGE ID`, если знаете его:

```shell
docker rmi abc12345def6
```

#### 5. Очистите неиспользуемые ресурсы Docker

Если вы хотите освободить место, очистите неиспользуемые Docker-ресурсы (контейнеры, сети, образы, тома):

```shell
docker system prune -a
```

Эта команда:

- Удаляет все неиспользуемые контейнеры.
- Удаляет все неиспользуемые образы.
- Очищает кэш Docker.

#### 6. Проверьте, что всё удалено

Убедитесь, что контейнер, данные и образы удалены:

```shell
docker ps -a       # Проверка контейнеров
docker images      # Проверка образов
ls /mnt/data/.HA   # Проверка данных Home Assistant
```

Если всё удалено корректно, вы не должны видеть никаких следов Home Assistant.

## Обновление Home Assistant

Остановка контейнера Home Assistant:

```shell
docker stop homeassistant
```

Удаление контейнера Home Assistant:

```shell
docker rm homeassistant
```

Удалите все образы Home Assistant:

```shell
docker rmi ghcr.io/home-assistant/home-assistant:stable
```

Запустите Home Assistant:

```shell
docker run -d --name homeassistant --privileged --restart=unless-stopped -e TZ=Europe/Moscow -v /run/dbus:/run/dbus:ro -v /mnt/data/.HA:/config --network=host ghcr.io/home-assistant/home-assistant:stable
```

# Временно решения для ошибки wb2504 wb-engine (не появляются утсройства в HA, после перезапуска контроллера)
Для исправления данной проблемы, я предлагаю создать сервис который будет перезапускать правила второй раз после запуса системы через 60сек

#### 1. Создайте systemd-юнит
Откройте или создайте файл:
```shell
nano /etc/systemd/system/restart-wb-rules-delayed.service
```

Вставьте следующее содержимое:
```shell
[Unit]
Description=Restart wb-rules service with delay after boot
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c "/usr/bin/sleep 60 && /usr/bin/systemctl restart wb-rules.service"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

#### 2. Включите юнит в автозагрузку
Введите последовательно(каждую строчку) данные команды:
```shell
systemctl daemon-reload
systemctl enable restart-wb-rules-delayed.service
```

#### 3. (Опционально) Проверка
После настройки вы можете проверить статус юнита:
```shell
systemctl status restart-wb-rules-delayed.service
```

Или протестировать его принудительно:
```shell
systemctl start restart-wb-rules-delayed.service
```
