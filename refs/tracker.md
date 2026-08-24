# Tracker refs

## Credentials

Предпочтительно: `~/.cursor/secrets/yandex-tracker.env`  
(`TRACKER_TOKEN_READ`, `TRACKER_CLOUD_ORG_ID`; для записи — `TRACKER_TOKEN_WRITE`).

Заголовок API: `X-Cloud-Org-ID` (не `X-Org-ID`).

## Проект ЯМ

| Поле | Значение |
|------|----------|
| INI | _(заполнить)_ |
| Project id | _(заполнить)_ |
| URL | _(заполнить)_ |

## Шаблоны YQL

```text
Queue: INI AND Summary: Яндекс
```

```text
Project: <id>
```

## Запись

Read-only по умолчанию. Create/update/comment — только по явной просьбе пользователя в сообщении.
