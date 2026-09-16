# Практическая работа: Развертывание CMS Joomla в Docker Compose

**Выполнил студент:** Абрамов Даниил Сергеевич

---

## 1. Подготовка окружения и конфигурация

Для выполнения работы была создана изолированная директория `joomla-docker`. В ней был подготовлен файл оркестрации `compose.yaml`, описывающий два взаимосвязанных сервиса: базу данных MariaDB и веб-сервер с Joomla.

```yaml
services:
  db:
    image: mariadb:11.5.2
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: example_root_password
      MYSQL_DATABASE: joomla_db
      MYSQL_USER: joomla_user
      MYSQL_PASSWORD: joomla_password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - joomla-network

  joomla:
    depends_on:
      - db
    image: joomla:latest
    ports:
      - "8082:80"
    restart: unless-stopped
    environment:
      JOOMLA_DB_HOST: db:3306
      JOOMLA_DB_USER: joomla_user
      JOOMLA_DB_PASSWORD: joomla_password
      JOOMLA_DB_NAME: joomla_db
    volumes:
      - joomla_data:/var/www/html
    networks:
      - joomla-network

networks:
  joomla-network:

volumes:
  db_data:
  joomla_data:
```

## 2. Сборка и запуск контейнеров

Запуск описанной инфраструктуры производился в терминале с помощью команды, переводящей сервисы в фоновый режим:
```bash
docker compose up -d
```

*Процесс загрузки образов и успешный старт контейнеров:*

![Запуск Docker Compose](https://github.com/Focsi4ek/my-mfua-notes-/blob/main/joomla-docker/4.png.png?raw=true)

## 3. Процесс веб-установки Joomla

После того как контейнеры перешли в статус `Started`, установка была продолжена через веб-интерфейс по адресу `http://localhost:8082`.

*Стартовая страница мастера установки:*

![Мастер установки Joomla]()

### Настройка учетной записи администратора
На данном этапе были заданы логин, пароль (согласно требованиям безопасности — более 12 символов) и email администратора:

![Параметры учетной записи](https://github.com/Focsi4ek/my-mfua-notes-/blob/main/joomla-docker/5.png.png?raw=true)

### Настройка подключения к базе данных


![](https://github.com/Focsi4ek/my-mfua-notes-/blob/main/joomla-docker/1.png.png?raw=true)

**Решение:** Для успешной связи между контейнерами в поле «Имя сервера баз данных» был указан алиас из файла `compose.yaml` — `db`. Настройки были приведены к следующему виду:
* **Хост:** `db`
* **Пользователь:** `joomla_user`
* **Имя БД:** `joomla_db`

## 4. Успешное завершение установки

После применения корректных параметров базы данных мастер установки успешно завершил работу. 

![Успешная установка](https://github.com/Focsi4ek/my-mfua-notes-/blob/main/joomla-docker/2.png.png?raw=true)

Инфраструктура развернута корректно, сайт и панель управления администратора функционируют штатно.

## 5. Управление и полезные команды

Находясь в папке `joomla-docker`

1. Просмотр логов приложения **Joomla** в реальном времени

```bash
docker compose logs -f joomla
```

2. Просмотр логов базы данных **db** в реальном времени

```bash
docker compose logs -f db
```

3. Приостановить запущенный контейнер:

```bash
docker compose stop
```

4. Запустить приостановленный контейнер:

```bash
docker compose start
```

5. Перезапустить:

```bash
docker compose restart
```

6. Показать конфигурацию текущего проекта:

```bash
docker compose config
```
