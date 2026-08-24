# Tracker refs

## Credentials

Предпочтительно: `~/.cursor/secrets/yandex-tracker.env`  
(`TRACKER_TOKEN_READ`, `TRACKER_CLOUD_ORG_ID`; для записи — `TRACKER_TOKEN_WRITE`).

Заголовок API: `X-Cloud-Org-ID` (не `X-Org-ID`).

## Проект ЯМ

| Поле | Значение |
|------|----------|
| INI | [INI-36](https://tracker.yandex.ru/INI-36) |
| Project id | 747 |
| URL | [проект](https://tracker.yandex.ru/pages/projects/747) · [задачи](https://tracker.yandex.ru/pages/projects/747/issues) |

Канон навигации: `context/03-tracker.md`.

## Шаблоны YQL

```text
Queue: INI AND Summary: Яндекс
```

```text
Project: 747
```

## Запись

Read-only по умолчанию. Create/update/comment — только по явной просьбе пользователя в сообщении.
