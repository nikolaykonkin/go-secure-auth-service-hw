# Практическое задание: Безопасный сервис аутентификации — Конкин Николай

Форк оригинального шаблона: [netology-code/goprod](https://github.com/netology-code/goprod)
Репозиторий с решением: [nikolaykonkin/go-secure-auth-service-hw](https://github.com/nikolaykonkin/go-secure-auth-service-hw)

## 🎯 Цель задания

Разработать безопасный REST API сервис с функциями регистрации и аутентификации пользователей на Go.

## 📋 Реализованный функционал

- ✅ Регистрация пользователя с хешированием пароля (bcrypt)
- ✅ Вход в систему с выдачей JWT токена
- ✅ Защищённый эндпоинт для получения профиля (требует JWT)
- ✅ Защита от SQL-инъекций (параметризованные запросы)

### API эндпоинты

| Метод | Путь        | Описание                 | Требует токен |
| ----- | ----------- | ------------------------ | ------------- |
| POST  | `/register` | Регистрация пользователя | Нет           |
| POST  | `/login`    | Вход в систему           | Нет           |
| GET   | `/profile`  | Получить профиль         | **Да**        |
| GET   | `/health`   | Проверка состояния       | Нет           |

## 🏗️ Структура проекта

```
├── main.go              # Главный файл с запуском сервера
├── handlers.go          # HTTP обработчики (реализовано)
├── models.go            # Структуры данных
├── database.go          # Работа с БД (реализовано)
├── auth.go              # JWT и bcrypt (реализовано)
├── middleware.go        # Проверка токена (реализовано)
├── docker-compose.yml   # PostgreSQL в Docker
├── init.sql             # Схема БД
├── .env.example          # Пример конфигурации
├── go.mod                # Зависимости
├── img/                   # Скриншоты результатов тестирования
└── README.md             # Этот файл
```

## 🔒 Как выполнены требования безопасности

### 1. Пароли хешируются bcrypt (`auth.go`)

```go
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return "", fmt.Errorf("failed to hash password: %v", err)
    }
    return string(bytes), nil
}
```

Пароль нигде в коде не сохраняется и не логируется в открытом виде — в БД записывается только `passwordHash`.

### 2. SQL запросы параметризованы (`database.go`)

Все запросы используют плейсхолдеры `$1, $2...`, нигде не применяется `fmt.Sprintf` для построения SQL:

```go
query := `INSERT INTO users (email, username, password_hash) VALUES ($1, $2, $3) RETURNING id, created_at`
err := db.QueryRow(query, email, username, passwordHash).Scan(&user.ID, &user.CreatedAt)
```

То же самое для `GetUserByEmail`, `GetUserByID`, `UserExistsByEmail` — во всех четырёх функциях запросы параметризованы.

### 3. JWT токены проверяются (`middleware.go`)

Эндпоинт `/profile` подключён через `AuthMiddleware`:

```go
http.HandleFunc("/profile", AuthMiddleware(ProfileHandler))
```

`AuthMiddleware` проверяет наличие заголовка `Authorization`, формат `Bearer <token>`, валидирует подпись токена (проверка алгоритма HMAC) и только после этого передаёт управление `ProfileHandler`. Без токена или с невалидным токеном возвращается `401 Unauthorized`.

## 🚀 Быстрый старт

```bash
# Настройка окружения
cp .env.example .env
# Изменить JWT_SECRET в .env на свой ключ (минимум 32 символа)

# Запуск базы данных
docker-compose up -d

# Установка зависимостей и запуск
go mod download
go run *.go
```

## ✅ Результаты тестирования

Все пункты чек-листа из задания проверены локально — ниже приведены скриншоты, подтверждающие работоспособность каждого требования.

### 1. Проверка состояния сервиса

Сервер запущен и подключение к БД активно:

![Проверка health-эндпоинта](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/01-health-check.png)

### 2. Регистрация пользователя

Запрос `POST /register` создаёт пользователя и возвращает JWT токен вместе с данными пользователя (без `password_hash` — поле исключено из JSON тегом `json:"-"`):

![Ответ на регистрацию](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/02-register-response.png)

### 3. Пароль сохранён как bcrypt-хеш, а не в открытом виде

Прямая проверка через `psql` — в столбце `password_hash` виден хеш, начинающийся с `$2a$` или `$2b$`, самого пароля в БД нет:

![Хеш пароля в базе данных](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/03-password-hash-in-db.png)

### 4. Вход в систему

Запрос `POST /login` с верными учётными данными возвращает валидный JWT токен:

![Ответ на вход в систему](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/04-login-response.png)

### 5. Токен корректно декодируется на jwt.io

Токен из ответа `/login`, вставленный на [jwt.io](https://jwt.io), содержит `user_id`, `email` и `username`:

![Декодированный JWT токен на jwt.io](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/05-jwt-io-decoded.png)

### 6. Запрос `/profile` без токена → 401

Защищённый эндпоинт отклоняет запрос без заголовка `Authorization`:

![Профиль без токена — 401 Unauthorized](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/06-profile-unauthorized.png)

### 7. Запрос `/profile` с валидным токеном → 200

С правильным токеном в заголовке `Authorization: Bearer <token>` эндпоинт возвращает данные профиля:

![Профиль с валидным токеном](https://github.com/nikolaykonkin/go-secure-auth-service-hw/blob/main/img/07-profile-authorized.png)

## 📝 Чек-лист перед сдачей

- [x] PostgreSQL запускается через `docker-compose up`
- [x] Приложение подключается к БД и не падает
- [x] Регистрация создаёт пользователя в БД
- [x] Пароли хранятся как bcrypt хеш, НЕ в открытом виде
- [x] Вход возвращает валидный JWT токен
- [x] Токен декодируется на https://jwt.io
- [x] Эндпоинт `/profile` требует токен (без токена → 401)
- [x] Эндпоинт `/profile` работает с правильным токеном
- [x] **ВСЕ** SQL запросы используют параметры `$1, $2...`
- [x] В коде НЕТ `fmt.Sprintf` для построения SQL
