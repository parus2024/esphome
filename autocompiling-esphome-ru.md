# Инструкция по автокомпилированию YAML-файлов в Detached-контейнерах ESPHome

**Автор:** Parus  
**Дата:** 27 октября 2025 г.  
**Версия:** 1.0  

---

## Введение
Это подробное руководство по настройке и использованию автокомпилирования YAML-файлов для ESPHome в detached-контейнерах (запускаемых с флагом `-d`). Оно подходит для работы с веб-интерфейсом ESPHome (доступным по портам вроде 2575 для версии 2025.7.5). Поскольку контейнеры detached, CLI-команда `esphome` недоступна напрямую в shell LXC — её нужно выполнять внутри контейнера с помощью `docker exec`. Мы также добавим скрипт для автоматизации, советы по локали, фоновому запуску и дополнительным действиям. Всё по шагам для удобства!

**Примечание:** Все команды и скрипты предназначены для выполнения в среде LXC на Proxmox или аналогичной. Убедитесь, что у вас есть права root.

---

## 1. Настройка Detached-контейнеров ESPHome
Detached-контейнеры позволяют запускать ESPHome в фоне, с доступом через веб-UI. Имя контейнера обычно соответствует версии, например, `esphome-2025-7-5` для версии 2025.7.5.

- **Проверка запущенных контейнеров:**  
  Запустите `docker ps` в LXC. Контейнер должен быть в списке (если нет, запустите его), например:  
  ```bash
  docker run -d --name esphome-2025-7-5 -p 2575:6052 -v /root/esphome-projects-2025.7.5:/config esphome/esphome:2025.7.5
  ```

- **Доступ к веб-интерфейсу:**  
  Откройте браузер и перейдите на `http://<IP-LXC-контейнера>:2575` (порт зависит от версии). Здесь можно загружать/редактировать YAML-файлы.

---

## 2. Рекомендуемый Способ Компиляции: Использование `docker exec`
Это позволяет компилировать файлы внутри работающего контейнера без остановки или входа в интерактивный режим.

1. **Выберите контейнер:** Убедитесь, что контейнер запущен (например, `esphome-2025-7-5`). Проверьте командой `docker ps`.
2. **Запустите компиляцию:**  
   ```bash
   docker exec -w /config esphome-2025-7-5 esphome compile test.yaml
   ```  
   - `-w /config`: Устанавливает рабочую директорию (где смонтирована папка `/root/esphome-projects-2025.7.5`).  
   - Замените `test.yaml` на имя вашего файла.  
   - Для подробного вывода добавьте `--verbose`:  
     ```bash
     docker exec -w /config esphome-2025-7-5 esphome compile --verbose test.yaml
     ```
3. **Проверьте результат:** После компиляции в LXC проверьте:  
   ```bash
   ls -la /root/esphome-projects-2025.7.5/.esphome/build/test/.pioenvs/test/firmware.bin
   ```
   Если файл есть — компиляция успешна.

---

## 3. Альтернативные Способы Компиляции
- **Через веб-интерфейс:** Откройте браузер на `http://<IP-LXC>:2575`, загрузите YAML, отредактируйте и скомпилируйте через UI. Удобно для тестов, но если LXC без браузера — используйте port-forwarding или SSH-tunnel из хоста Proxmox.
- **Интерактивный вход в контейнер:** Для полного доступа войдите:  
  ```bash
  docker exec -it esphome-2025-7-5 bash
  ```
  затем `cd /config && esphome compile alarm-21.yaml`, выйдите с `exit`.
- **Автоматизация нескольких версий:** Используйте Bash-скрипт с циклом, который запускает `docker exec` для каждой директории (пример ниже).

**Структура директорий в /root:**  
```
/root/
  esphome-projects-2025.7.5/
    firmware.yaml
    sensor.yaml
  esphome-projects-2024.12.10/
    main.yaml
```  
Скрипт обработает папки, начинающиеся с `esphome-projects-`, и найдёт в них `.yaml`-файлы.

---

## 4. Установка Локали UTF-8 для Корректного Отображения Кириллицы
Если в nano кириллица отображается некорректно, настройте локаль внутри контейнера.

1. **Генерация и установка локали:**  
   ```bash
   locale-gen ru_RU.UTF-8
   dpkg-reconfigure locales
   ```  
   Или запустите `dpkg-reconfigure locales` и выберите `ru_RU.UTF-8` (или `en_US.UTF-8`).
2. **Установка пакета, если нужно:**  
   ```bash
   apt update
   apt install locales
   ```
3. **Настройка переменных окружения:**  
   ```bash
   echo "export LANG=ru_RU.UTF-8" >> ~/.bashrc
   echo "export LC_ALL=ru_RU.UTF-8" >> ~/.bashrc
   source ~/.bashrc
   ```
4. **Проверка:**  
   ```bash
   locale
   ```
   Должен отобразиться `LANG=ru_RU.UTF-8` и `LC_ALL=ru_RU.UTF-8`. Теперь кириллица в nano будет работать правильно.

---

## 5. Запуск Скрипта в Фоновом Режиме
Стандартный Bash-скрипт остановится при закрытии консоли. Чтобы он работал в фоне:

- **Простой фон с `&`:**  
  ```bash
  ./ваш_скрипт.sh &
  ```
  Командная строка освободится сразу.
- **С `nohup` (для продолжения при закрытии терминала):**  
  ```bash
  nohup ./ваш_скрипт.sh > /путь/к/логам/mylog.out 2>&1 &
  ```
  - Вывод сохранится в `mylog.out`.  
  - Для вашего скрипта:  
    ```bash
    nohup ./compile_esphome_2025_7_5.sh > /root/esphome_compile.log 2>&1 &
    ```
- **С `tmux` или `screen`:** Создаёт виртуальный терминал.  
  - Установите: `apt install tmux` или `apt install screen`.  
  - Запустите: `tmux` (или `screen`), затем `./ваш_скрипт.sh`, отключитесь (`Ctrl+B D` в tmux), скрипт продолжит работать.

---

## 6. Итоговый Скрипт `compile_esphome_2025_7_5.sh`
Вот готовый скрипт для автоматизации. Он обрабатывает YAML-файлы в директории, пропускает `secrets.yaml`, проверяет наличие `firmware.bin` и компилирует при необходимости. Скопируйте код ниже в файл.

```bash
#!/bin/bash

# ================================
# Настройка переменных
# ================================
# Вписать нужную версию esphome
VERSION="2025.7.5"
DIR="/root/esphome-projects-${VERSION}"
CONTAINER_NAME="esphome-${VERSION//./-}"
DOCKER_CONFIG_PATH="/config"
LOG_FILE="/root/esphome-projects-${VERSION}/compile_esphome_${VERSION//./_}.log"

# Очистка лог-файла в начале
: > "$LOG_FILE"

# Вспомогательная функция для записи в лог
log() {
  echo "$*" | tee -a "$LOG_FILE"
}

# ================================
# Время начала выполнения
# ================================
start_time=$(date +%s)
script_start_time=$(date)

# ================================
# Инициализация счетчиков и массивов
# ================================
processed=0
skipped=0
errors=0
declare -a error_files=()

# ================================
# Проверки
# ================================

if [ ! -d "$DIR" ]; then
  log "ПАПКА $DIR НЕ НАЙДЕН!"
  exit 1
fi

if ! docker ps --format="{{.Names}}" | grep -q "^$CONTAINER_NAME$"; then
  log "КОНТЕЙНЕР \"$CONTAINER_NAME\" НЕ ЗАПУЩЕН!"
  exit 1
fi

# ================================
# Обработка файлов
# ================================
shopt -s lastpipe

mapfile -t yaml_files < <(find "$DIR" -maxdepth 1 -name "*.yaml" -printf "%f\n")
if [ ${#yaml_files[@]} -eq 0 ]; then
  log "Не найдено ни одного YAML-файла в $DIR."
  exit 1
fi

for filename in "${yaml_files[@]}"; do
  log "----------------------------------------"
  log "Обработка файла: $filename"
  ((processed++))

  if [[ "$filename" == "secrets.yaml" ]]; then
    log "Пропускаю $filename — это файл секретов."
    continue
  fi
  base_name="${filename%.yaml}"
  bin_path_in_container="/config/.esphome/build/$base_name/.pioenvs/$base_name/firmware.bin"
  log "Найденные файлы .bin: $bin_path_in_container"

  if docker exec "$CONTAINER_NAME" test -f "$bin_path_in_container"; then
    mod_time=$(docker exec "$CONTAINER_NAME" stat -c %Y "$bin_path_in_container")
    mod_time_readable=$(docker exec "$CONTAINER_NAME" date -d "@$mod_time" '+%Y-%m-%d %H:%M:%S')
    log "Файл $bin_path_in_container уже существует. Пропуск сборки для $filename. Время последней модификации: $mod_time_readable"
    ((skipped++))
    continue
  fi

  log "Место, куда кладется .bin: $bin_path_in_container"

  # копирование
  log "КОПИРУЮ $filename в контейнер..."
  if docker cp "$DIR/$filename" "$CONTAINER_NAME":"$DOCKER_CONFIG_PATH/"; then
    log "Успешное копирование."
  else
    log "Ошибка при копировании файла $filename."
    ((errors++))
    error_files+=("$filename")
    continue
  fi

  # компиляция
  log "Компиляция $filename внутри контейнера..."
  if docker exec -w "$DOCKER_CONFIG_PATH" "$CONTAINER_NAME" esphome compile "$filename"; then
    log "✅ УСПЕШНО: $filename (компиляция завершена: $(date))"
  else
    log "❌ Ошибка при компиляции $filename (ошибка: $(date))"
    ((errors++))
    error_files+=("$filename")
  fi
done

# ================================
# Время окончания выполнения и суммарное время
# ================================
end_time=$(date +%s)
script_end_time=$(date)

duration=$((end_time - start_time))
minutes=$((duration / 60))
seconds=$((duration % 60))

log "========================================"
log "Обработка завершена!"
log "Обработано файлов: $processed"
log "Пропущено (уже есть .bin): $skipped"
log "Ошибок при компиляции: $errors"

if [ $errors -gt 0 ]; then
  echo "Файлы с ошибками:" | tee -a "$LOG_FILE"
  for f in "${error_files[@]}"; do
    echo "- $f" | tee -a "$LOG_FILE"
  done
fi

log "Общее время: ${duration} секунд (${minutes} минут и ${seconds} секунд)."
log "Начало: $script_start_time"
log "Окончание: $script_end_time"
log "Полный лог см. по адресу: $LOG_FILE"

#cat "$LOG_FILE" # Вывод лог-файла в консоль
```

---

## 7. Работа со Скриптом
- **Редактирование:** `nano /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.
- **Просмотр лога:** `cat /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.log`.
- **Запуск:**  
  - Дайте права: `chmod +x /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.  
  - Запустите: `bash /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh` (или `./compile_esphome_2025_7_5.sh`).

---

## 8. Дополнительные Действия
- **Путь к скомпилированному bin-файлу:** `/root/esphome-projects-2025.7.5/.esphome/build/alarm-33/.pioenvs/alarm-33/firmware.bin` (замените на имя файла).
- **Удаление папки:** `rm -rf /root/esphome-projects-2024-6-0/`.
- **Удаление скрипта:** `rm /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.
- **Проверка наличия bin-файла:**  
  ```bash
  docker exec esphome-2025-7-5 test -f "/config/.esphome/build/mirror-ld2420/.pioenvs/mirror-ld2420/firmware.bin" && echo "Есть" || echo "Нет"
  ```
- **Вход по PuTTY (SSH) в контейнер:**  
  1. Установите SSH: `apt update && apt install openssh-server`.  
  2. Включите: `systemctl enable ssh && systemctl start ssh`.  
  3. Проверьте статус: `service sshd status`.  
  4. Отредактируйте `/etc/ssh/sshd_config` (раскомментируйте/добавьте: `PasswordAuthentication yes`, `PermitRootLogin yes`).  
  5. Перезапустите: `service ssh restart`.  
     Теперь подключайтесь по PuTTY к IP LXC с root-паролем.
