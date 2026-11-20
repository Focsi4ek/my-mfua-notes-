# 📓 Мои заметки по CI/CD

> Конспект по парам + разбор своих лабораторных.
> Дата: ноябрь 2026

---

## 🗓️ Пара 1. Что такое CI/CD

### CI — Continuous Integration
Разбирали на паре. Смысл: **каждый push автоматически проверяется**. Сразу собирается, гоняются линтеры и тесты. Ошибку видно через 2 минуты, а не через неделю.

Записал себе главное:
- Ошибка ловится **до** merge в main.
- Все работают по одним правилам (один линтер, один форматтер).
- Сборка воспроизводима — «у меня работает» больше не отмазка.

### CD — Continuous Delivery / Deployment
- **Delivery** — после CI артефакт готов, но публикует человек.
- **Deployment** — публикация автоматическая.

У меня в лабах CD — это три вещи:
- **Pages** (`hello-pages`) — сайт обновляется сам после push.
- **GHCR** (`hello-go`, `hello-rust`, `my-fullstack-app`) — образ уходит в реестр.
- **Releases** (`hello-go-bin`, `hello-python`, `hello-dotnet`) — бинарники по тегу `v*`.

### Классический пайплайн
```
push → lint → test → build → (release / deploy)
```
⚠️ **Запомнить:** каждый шаг — ворота. Упал линтер — тесты не запустятся. Упали тесты — образ не соберётся.

---

## 🗓️ Пара 2. GitHub Actions: из чего состоит workflow

### Что где лежит
Workflow — это YAML-файл в `.github/workflows/`. У меня в каждом проекте свой: `ci.yml`, `deploy.yml`, `hello.yml`.

### Словарь (выписал с доски)

| Термин | Что это |
|:---|:---|
| **Workflow** | Один пайплайн = один YAML-файл |
| **Event** | Триггер: `push`, `pull_request`, `schedule`, `workflow_dispatch`, тег `v*` |
| **Job** | Группа шагов на отдельной машине. Jobs по умолчанию параллельны |
| **Step** | Один шаг: либо `run:` (команда), либо `uses:` (готовый action) |
| **Runner** | Машина: `ubuntu-latest`, `macos-latest`, `windows-latest` |
| **Action** | Готовый блок, например `actions/checkout@v4` |
| **Matrix** | Запуск одного job с разными параметрами |
| **Artifact** | Файл из workflow (архив образа, отчёт). Хранится 90 дней |
| **Secret** | Зашифрованная переменная (токены, пароли) |
| **Service Container** | Контейнер-зависимость (например, Postgres) для тестов |
| **needs** | «Job B стартует только после успеха job A» |

### Минимальный workflow (переписал с пары)
```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./...
```

⚠️ **Важно:** без `actions/checkout` файлов репозитория на runner нет — сразу ошибка «file not found».

### Триггеры — как читать `on:`
```yaml
on:
  push:
    branches: [main]
    tags: ['v*']          # только теги вида v1.0.0
  pull_request:
    branches: [main]
  workflow_dispatch:       # кнопка «Run workflow» вручную
```

### Матрицы
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    go: ['1.21', '1.22']
```
Это **6 jobs** (3 ОС × 2 версии). Если один упал — весь workflow красный. `fail-fast: false` даёт остальным доработать.

Где у меня:
- `my-python-app` — Python 3.9–3.12
- `my-node-app` — Node.js 18, 20, 22
- `hello-go-bin`, `hello-python`, `hello-dotnet` — ОС × архитектуры

### Артефакты
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: coverage.xml
```
Скачиваются из **Actions → запуск → Artifacts**.

У меня:
- Архивы Docker-образов — `my-cpp-app`, `my-php-app`, `my-rust-app`
- Отчёт покрытия — `my-go-app`

### Кэш (ускоряет повторные запуски)
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/go-build
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
```

### Контексты, которые точно пригодятся
- `${{ github.sha }}` — хеш коммита
- `${{ github.ref_name }}` — имя ветки/тега
- `${{ secrets.GITHUB_TOKEN }}` — автотокен, доступен всегда
- `${{ matrix.os }}` — текущее значение матрицы

---

## 🗓️ Пара 3. Docker в CI

### Зачем контейнеры в CI
- **Воспроизводимость** — одинаковое окружение везде.
- **Изоляция** — Postgres для тестов не конфликтует с локальным.
- **Переносимость** — образ, собранный в CI, запускается где угодно.

### Dockerfile — multi-stage build
```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY . .
RUN go build -o /app

FROM alpine:3.19
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```
⚠️ **Запомнить:** два `FROM` = multi-stage. Первый собирает, второй содержит только бинарник → образ маленький.

### Основные команды
```bash
docker build -t my-app .          # собрать
docker run --rm my-app            # запустить и удалить
docker run -p 8080:8080 my-app    # пробросить порт
docker images                     # образы
docker ps                         # запущенные
```

### Docker Compose
```yaml
services:
  app:
    build: .
    ports: ["8000:8000"]
    depends_on: [db]
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
```
У меня используется в `my-fullstack-app` для локального запуска.

### Service Container в CI (для тестов с реальной БД)
```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: postgres
    ports: ['5432:5432']
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
```
⚠️ **Зачем `--health-cmd`:** чтобы job не начал тесты раньше, чем БД готова принимать подключения.

Так сделано в `my-fullstack-app`.

---

## 🗓️ Пара 4. Линтеры и тесты — по языкам

Выписал таблицу, чтобы не путаться:

| Язык | Линтер / форматтер | Тесты | Где у меня |
|:---|:---|:---|:---|
| Python | Ruff, Flake8 | pytest | my-python-app, my-fullstack-app |
| JavaScript/Node | ESLint | Jest | my-node-app |
| TypeScript/React | oxlint, tsc | Vitest | hello-pages |
| Go | gofmt, go vet, golangci-lint | go test | my-go-app, hello-go |
| Rust | rustfmt, Clippy | cargo test | my-rust-app, hello-rust |
| PHP | php -l (синтаксис) | PHPUnit | my-php-app |
| C++ | clang-format | Google Test (CTest) | my-cpp-app |
| Java | Maven checks | JUnit 5 | hello-java |
| C#/.NET | dotnet format | xUnit | hello-dotnet |

Определения (записал с пары):
- **Линтер** — ищет ошибки и потенциальные баги без запуска кода.
- **Форматтер** — приводит код к одному виду.
- **Тест** — запускает код и сверяет результат с ожидаемым.
- **Покрытие (coverage)** — % строк, которые выполнились во время тестов. В `my-go-app` сохраняется как артефакт.

---

## 🗓️ Пара 5. Публикация — три способа

### Способ 1: GHCR (GitHub Container Registry)
Реестр Docker-образов, встроенный в GitHub.

```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: docker/build-push-action@v6
  with:
    push: true
    platforms: linux/amd64,linux/arm64
    tags: ghcr.io/focsi4ek/hello-go:latest,ghcr.io/focsi4ek/hello-go:${{ github.sha }}
```

⚠️ **Ключевое:**
- `GITHUB_TOKEN` — встроенный, настраивать не надо.
- **Buildx + QEMU** — позволяют собрать образ под arm64 на amd64-runner (кросс-сборка).
- Теги `latest` + SHA — можно откатиться к конкретному коммиту.

У меня: `hello-go`, `hello-rust`, `my-fullstack-app`.

### Способ 2: GitHub Releases (бинарники)
```yaml
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go build -o hello-${{ matrix.os }}-${{ matrix.arch }}
      - uses: softprops/action-gh-release@v2
        with:
          files: hello-*
```

⚠️ **Ключевое:**
- Триггер — только тег `v*`, обычный push релиз не запускает.
- Версия внедряется в бинарник: `-ldflags "-X main.version=${{ github.ref_name }}"`.
- **Self-contained** — не требует SDK (PyInstaller, .NET self-contained).
- **Проверка запуском** — после сборки workflow запускает бинарник на нужном runner (`hello-python`, `hello-dotnet`).

У меня: `hello-go-bin`, `hello-python`, `hello-dotnet`.

### Способ 3: GitHub Pages (статика)
```yaml
- uses: actions/configure-pages@v5
- uses: actions/upload-pages-artifact@v3
  with:
    path: dist/
- uses: actions/deploy-pages@v4
```

⚠️ **Ключевое:**
- Нужны разрешения: `pages: write`, `id-token: write`.
- `dist/` — результат сборки (`vite build` для React).
- Сайт живёт по адресу `https://<user>.github.io/<repo>/`.

У меня: `hello-pages`.

### Сравнение (выписал для себя)

| | GHCR | Releases | Pages |
|:---|:---|:---|:---|
| Что публикует | Docker-образ | Бинарники | Статику |
| Кто потребляет | Сервер / Kubernetes | Пользователь скачивает | Браузер |
| Триггер | push в main | тег `v*` | push в main |
| Доступ | `docker pull` | страница релиза | URL сайта |

---

## 🗓️ Пара 6. Git — то, что спросят точно

### Три состояния файла
1. **Working directory** — то, что редактирую.
2. **Staging area** — то, что добавил через `git add`.
3. **Repository** — то, что зафиксировал через `git commit`.

### Основные команды
```bash
git status                # что изменено
git add <файл>            # в staging
git commit -m "..."       # зафиксировать
git log --oneline         # история
git diff                  # что изменилось
git branch                # ветки
git checkout -b feature   # создать и переключиться
```

### Удалённые репозитории
- **origin** — мой форк.
- **upstream** — исходный репозиторий (то, с чем синхронизируюсь).

```bash
git remote -v                    # список
git fetch upstream               # скачать, не применяя
git merge upstream/master        # слить
git push origin main             # отправить
```

⚠️ **Из практики:** у преподавателя на GitFlic основная ветка — `master`, а не `main`. Поэтому `git merge upstream/main` падал с `not something we can merge`. Надо смотреть вывод `git fetch upstream` — там список веток.

### Merge vs Rebase
- **Merge** — создаёт коммит слияния. История ветвистая, но полная. Безопасно.
- **Rebase** — «перекладывает» мои коммиты поверх чужих. История линейная. Чище, но переписывает хеши → потом может понадобиться `push --force`.

### Конфликты
Git вставляет маркеры:
```
моя версия
```
Оставляю нужное, удаляю маркеры, потом:
- при merge: `git add` → `git commit`
- при rebase: `git add` → `git rebase --continue`

### Ошибки, которые я уже видел
| Ошибка | Что значит | Что делать |
|:---|:---|:---|
| `'upstream' does not appear to be a git repository` | upstream не добавлен | `git remote add upstream <url>.git` |
| `upstream/main - not something we can merge` | Нет ветки `main` | Смотреть `git fetch`, у препода `master` |
| `! [rejected] (fetch first)` | На GitHub есть коммиты, которых нет локально | `git pull --rebase origin main` → `git push` |

---

## 🗓️ Пара 7. Разбор своих лаб — что чем закрываю

| Проект | Главный навык |
|:---|:---|
| `my-first-cicd` | Первый workflow: checkout + команда |
| `my-python-app` | Матрица Python + Docker |
| `my-node-app` | Матрица Node + Docker |
| `my-go-app` | Покрытие тестами как артефакт |
| `my-rust-app` | Debug + Release, образ в артефакте |
| `my-php-app` | Синтаксис + PHPUnit, образ в артефакте |
| `my-cpp-app` | CMake + CTest, образ в артефакте |
| `hello-java` | Maven, JAR + Docker-образ |
| `hello-pages` | Полный CI/CD: tsc → lint → test → build → Pages |
| `my-fullstack-app` | Service Container (Postgres), публикация в GHCR |
| `hello-go` | Buildx + QEMU, две архитектуры, GHCR |
| `hello-rust` | То же, но на Rust |
| `hello-go-bin` | Матрица 5 платформ, версия из тега, Releases |
| `hello-python` | PyInstaller onefile, проверка запуском |
| `hello-dotnet` | Self-contained single-file, проверка запуском |
