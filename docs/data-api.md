# DATA-API 1.0

Машиночитаемое описание обязательных HTTP-проверок для системы автоматизированной
технической оценки. Файл `DATA-API.yaml` в корне репозитория составлен по формату
DATA-API 1.0 и проходит эталонный валидатор (код завершения `0`).

## Состав

| Файл | Назначение |
|------|-----------|
| `DATA-API.yaml` | Сценарий обязательных проверок API |
| `openapi.yaml` | OpenAPI 3.1 спецификация всего HTTP API (источник: [api.md](api.md)) |

## Сценарий проверок

| id | Запрос | Роль | Что проверяет |
|----|--------|------|---------------|
| `health` | `GET /` | `public` | Приложение отвечает, отдаётся HTML |
| `list-decks` | `GET /api/decks` | `user` | Каталог колод, поля `filters`, `decks` |
| `start-cards` | `POST /api/cards/sessions` | `user` | Создание партии; `extract: sessionId ← $.id` |
| `read-cards` | `GET /api/cards/sessions/{sessionId}` | `user` | Чтение сессии, `dependsOn: start-cards` |
| `list-tests` | `GET /api/tests` | `user` | Список тестов, поле `tests` |
| `start-quiz` | `POST /api/tests/sessions` | `user` | Создание сессии теста; `extract: quizId ← $.id` |
| `read-quiz` | `GET /api/tests/sessions/{quizId}` | `user` | Чтение сессии, `dependsOn: start-quiz` |
| `admin-check` | `GET /admin/api/check` | `admin` | Пароль админки, ответ `{"ok": true}` |

Cleanup: `POST /api/cards/sessions/{sessionId}/exit` и
`POST /api/tests/sessions/{quizId}/leave` — API не предусматривает удаление сессий,
поэтому проверки завершают их максимально щадяще: партия уходит в паузу с сохранением
прогресса, тест завершается с частичным результатом. На статистику это влияет так же,
как обычное пользование мини-приложением.

## Роли и передача учётных данных

- `public` — без аутентификации.
- `user` — заголовок `X-Init-Data` со строкой данных запуска от MAX (HMAC-подпись
  токеном бота, действует 24 часа). В `DATA-API.yaml` передаётся как `${initData}`:
  реальное значение платформа технической проверки подставляет из защищённого канала.
- `admin` — заголовок `Authorization: Bearer <пароль>`, в файле — `Bearer ${adminPassword}`.
  Пароль задаётся переменной окружения `ADMIN_PASSWORD` и должен совпадать с тем,
  что передаётся платформе по защищённому каналу.

Секреты в `DATA-API.yaml` не хранятся — только ссылки на переменные защищённого канала.

## Проверка валидатором

```bash
python -m pip install -r requirements.txt   # из комплекта DATA-API (PyYAML, jsonschema)
python validate_data_api.py DATA-API.yaml --schema DATA-API.schema.json --openapi ./openapi.yaml
```

Путь `api.openapi` в файле указывает на `./openapi.yaml`, поэтому явный флаг `--openapi`
необязателен. Ожидаемый результат — код `0` и сообщение:

```
ПОДТВЕРЖДЕНИЕ файл DATA-API.yaml соответствует версии схемы 1.0
```

Валидатор выдаёт предупреждения вида «использует переменные, не извлечённые предыдущими
проверками» для `${initData}` и `${adminPassword}` — это штатно: значения приходят из
защищённого канала, а не из предыдущих шагов сценария. На код завершения они не влияют.
