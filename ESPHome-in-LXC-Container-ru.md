# Настройка ESPHome в Proxmox с помощью LXC-контейнера

Эта инструкция помогает настроить ESPHome в Proxmox VE. Мы используем LXC-контейнер (лёгкий контейнер на уровне ОС), чтобы обойти ограничения HAOS (где аддон ESPHome всегда на latest версии). Это позволяет экспериментировать с разными версиями ESPHome (например, 2023.12.0 или 2025.7.5), не затрагивая Home Assistant.

## Предварительные требования
- Proxmox VE установлен и запущен.
- Доступ к Proxmox Web UI (обычно https://<IP-хоста>:8006).
- Базовые знания Linux (команды apt, systemctl).
- Свободное место на хосте Proxmox (минимум 10-20 GB для контейнера + образы Docker).

## Почему LXC-контейнер?
- HAOS не позволяет устанавливать Docker напрямую (без рисков для стабильности).
- LXC изолирует ESPHome от HA, даёт гибкость с версиями и меньше ресурсов, чем полноценная VM.
- Мы можем запускать несколько контейнеров ESPHome с разными версиями на разных портах.

## Раздел 1: Создание LXC-контейнера в Proxmox

1. **Заходим в Proxmox Web UI:**  
   Открываем браузер и переходим на https://<IP-хоста>:8006 (заменяем на IP своего хоста).  
   Входим под root или администратором.

2. **Создаём новый контейнер:**  
   Выбираем свой узел (node) в левом меню.  
   Нажимаем **Create CT (Create Container)**.  
   - **General:**  
     - Hostname: ESPHome-Container (или любое имя).  
     - Password: Задаём пароль для root (запоминаем его).  
     - Unprivileged container: Оставляем галочку (для безопасности).  
   - **Template:**  
     Выбираем Ubuntu или Debian (скачиваем, если не загружено: нажимаем Templates и ищем).  
   - **Root Disk:**  
     - Storage: Выбираем local-lvm или другое хранилище с местом.  
     - Disk size: 10-20 GB (хватит для старта; можно расширить позже).  
   - **CPU:**  
     - Cores: 1-2 (достаточно для компиляции ESPHome).  
   - **Memory:**  
     - Memory: 1024-2048 MB (1-2 GB RAM).  
   - **Network:**  
     - Bridge: vmbr0 (или тот же, что у HAOS-VM, для доступа к сети и HA).  
     - IPv4: DHCP (автоматический IP) либо задаём постоянный адрес вручную.  
   Нажимаем **Create** и ждём завершения (1-5 минут).

3. **Запускаем контейнер:**  
   В списке контейнеров (CT) выбираем ESPHome-Container.  
   Нажимаем **Start**.  
   Проверяем статус: Должен быть "Running".

4. **Подключаемся к контейнеру:**  
   В Proxmox UI: Выбираем контейнер > Console (открывается терминал внутри).  
   Или по SSH (если сеть настроена): `ssh root@<IP-контейнера>` (IP проверяем в настройках сети контейнера в Proxmox UI).  
   Входим как root с паролем, который задали при создании.  
   *Пояснение: LXC использует ядро хоста, но изолирован. Теперь внутри Ubuntu/Debian.*

## Раздел 2: Установка Docker и подготовка

1. **Обновляем систему и устанавливаем Docker:**  
   В консоли контейнера (как root) выполняем по порядку:  
   - `apt update`: Обновляет список пакетов.  
   - `apt upgrade -y`: Обновляет систему (с -y пропускает подтверждения).  
   - `apt install docker.io -y`: Устанавливает Docker.  
   - `systemctl start docker && systemctl enable docker`: Запускает и включает автозапуск Docker.  
   - `usermod -aG docker root`: Добавляет root в группу Docker (чтобы не писать sudo для команд Docker).  
   Проверяем: `docker --version` — выводит версию (например, 24.x.x).

2. **Создаём папку для проектов:**  
   `mkdir ~/esphome-projects` (это /root/esphome-projects, если ты root).  
   Копируем свои YAML-файлы сюда, если надо, используем другие методы копирования. Также папки со шрифтами, изображениями, вложениями и файл secrets.  
   *Пояснение: Docker нужен для запуска контейнеров ESPHome. Папка ~/esphome-projects монтируется в контейнеры.*

## Раздел 3: Настройка ESPHome с разными версиями

ESPHome — это инструмент для прошивки ESP32/Arduino. Мы запускаем несколько Docker-контейнеров с разными версиями на уникальных портах.

1. **Скачиваем образы ESPHome:**  
   ```
   docker pull esphome/esphome:2023.12.0
   docker pull esphome/esphome:latest
   docker pull esphome/esphome:2024.6.0
   docker pull esphome/esphome:2025.7.5
   ```  
   (или другие теги с https://hub.docker.com/r/esphome/esphome/tags).  
   Проверяем: `docker images` — список загруженных образов.

2. **Запускаем контейнеры с версиями:**  
   Используем уникальные порты, чтобы избежать конфликтов. Для удобства делаем порты "похожими" на версию (например, 23120 для 2023.12.0).  
   Примеры команд (запускаем по одной в консоли контейнера):  
   - Для 2023.12.0 (порт 23120):  
     ```
     docker run -d --name esphome-2023-12-0 --restart unless-stopped -v ~/esphome-projects:/config -p 23120:6052 -p 23121:6123 esphome/esphome:2023.12.0
     ```  
   - Для latest (порт 6053):  
     ```
     docker run -d --name esphome-latest --restart unless-stopped -v ~/esphome-projects:/config -p 6053:6052 -p 6124:6123 esphome/esphome:latest
     ```  
   - Для 2024.6.0 (порт 2460):  
     ```
     docker run -d --name esphome-2024-6-0 --restart unless-stopped -v ~/esphome-projects:/config -p 2460:6052 -p 2461:6123 esphome/esphome:2024.6.0
     ```  
   - Для 2025.7.5 (порт 2575):  
     ```
     docker run -d --name esphome-2025-7-5 --restart unless-stopped -v ~/esphome-projects:/config -p 2575:6052 -p 2576:6123 esphome/esphome:2025.7.5
     ```  
   *Пояснение флагов:*  
   - `-d`: Запуск в фоне (detached).  
   - `--name`: Уникальное имя контейнера.  
   - `--restart unless-stopped`: Автозапуск при перезагрузке контейнера.  
   - `-v ~/esphome-projects:/config`: Монтирует папку проектов (файлы общими для всех контейнеров).  
   - `-p 2575:6052`: Пробрасывает порт (внешний:внутренний) для веб-интерфейса. Меняем внешний порт для каждого контейнера.  
   - Второй порт (например, 2576:6123) — для OTA/API, чтобы не конфликтовать.

3. **Проверяем и получаем доступ:**  
   `docker ps`: Список запущенных контейнеров (должны быть "Up").  
   В браузере: http://<IP-контейнера>:2575 (например, http://192.168.1.100:2575 для версии 2025.7.5). IP контейнера — тот же, что у хоста или проверяем в Proxmox.  
   Загружаем YAML в веб-интерфейс, компилируем и прошиваем ESP.

4. **Управляем контейнерами:**  
   - Останавливаем: `docker stop esphome-2025-7-5`  
   - Запускаем: `docker start esphome-2025-7-5`  
   - Логи: `docker logs esphome-2025-7-5`  
   - Входим внутрь: `docker exec -it esphome-2025-7-5 bash` (для ручных команд).  
   *Пояснение: Теперь у нас несколько версий ESPHome на разных портах. Проекты общие, но тестируем YAML прошивки на разных версиях.*

## Раздел 4: Настройка FTP для загрузки файлов

FTP позволяет загружать файлы (шрифты, изображения) в ~/esphome-projects. Мы устанавливаем vsftpd (простой FTP-сервер). Альтернативы: SCP/SFTP (без установки) или ручное копирование.

1. **Устанавливаем vsftpd:**  
   В консоли контейнера (как root):  
   ```
   apt update
   apt install vsftpd -y
   systemctl start vsftpd
   systemctl enable vsftpd
   ```

2. **Настраиваем конфигурацию:**  
   Делаем бэкап: `cp /etc/vsftpd.conf /etc/vsftpd.conf.backup`  
   Редактируем: `nano /etc/vsftpd.conf` (или vim).  
   Добавляем/изменяем в конце файла:  
   ```
   write_enable=YES
   local_enable=YES
   chroot_local_user=YES
   allow_writeable_chroot=YES
   user_sub_token=$USER
   local_root=/root/esphome-projects  # Если ты root; для обычного пользователя: /home/$USER/esphome-projects
   ```  
   - `write_enable=YES`: Разрешает запись (загрузку файлов).  
   - `local_enable=YES`: Доступ для системных пользователей.  
   - `chroot_local_user=YES`: Ограничивает пользователей их папкой.  
   - `allow_writeable_chroot=YES`: Позволяет запись в chroot.  
   - `local_root`: Устанавливает корневую папку на ~/esphome-projects.  
   Сохраняем (нажимаем Ctrl+O, Enter, Ctrl+X в nano) и перезапускаем: `systemctl restart vsftpd`.

3. **Создаём пользователя для FTP (рекомендуем, если не root):**  
   ```
   useradd -m esphomeftp
   passwd esphomeftp  # вводим пароль
   chown -R esphomeftp:esphomeftp ~/esphome-projects
   ln -s ~/esphome-projects /home/esphomeftp/projects  # символическая ссылка для доступа
   ```  
   В vsftpd.conf изменяем `local_root=/home/esphomeftp/projects`.

4. **Подключаемся FTP-клиентом:**  
   Используем FileZilla или другой, или командную строку: `ftp <IP-контейнера>` (логин: esphomeftp или root, пароль).  
   Загружаем файлы в папку (например, fonts/ или images/).  
   Если зависло: Выходим с `bye` или `quit`.

5. **Troubleshooting:**  
   - Логи: `journalctl -u vsftpd` или `/var/log/vsftpd.log`.  
   - Если "permission denied": Проверяем права (`chmod 770 ~/esphome-projects`, `chown root:esphomeftp ~/esphome-projects`).  
   - Firewall: `ufw allow 21/tcp` (если установлен).  
   - Для root: Добавляем `allow_root_login=YES` в vsftpd.conf (но лучше используем esphomeftp).

## Раздел 5: Расширение диска LXC-контейнера (если место закончилось)

1. **Через Proxmox Web UI:**  
   Выбираем контейнер > Resources > Disk > Resize.  
   Вводим новый размер (например, +10G) > Resize Disk.

2. **Обновляем файловую систему внутри контейнера:**  
   `df -h` (проверяем путь, например, /dev/mapper/pve-vm--101--disk--0).  
   `resize2fs /dev/mapper/pve-vm--101--disk--0` (для ext4; заменяем ID на свой, проверяем `lsblk`).

3. **Через CLI на хосте Proxmox:**  
   На хосте: `pct config <ID>` (<ID> контейнера).  
   `pct resize <ID> rootfs +10G`.  
   Затем обновляем ФС внутри, как выше. 
