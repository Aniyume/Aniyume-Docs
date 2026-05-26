# Regression checklist

## Назначение

Этот документ фиксирует минимальный и расширенный regression checklist перед и после крупных изменений backend/frontend/admin.

Цели:

- не сломать рабочие пользовательские сценарии;
- иметь единый список обязательных проверок после рефакторинга;
- подготовить базу для CI/smoke tests;
- использовать checklist для дипломной демонстрации и стабилизации проекта.

---

## 1. Основные правила использования

### Когда прогонять checklist

Обязательно прогонять:

- перед началом крупного рефакторинга;
- после изменений в auth;
- после изменений в catalog/episodes;
- после изменений в social/watch party;
- после изменений в admin;
- перед merge крупной ветки;
- перед демонстрацией/защитой.

### Формат фиксации

Для каждой проверки отмечать:

- Passed / Failed;
- что именно проверялось;
- при необходимости — скриншот, лог или ответ API.

---

## 2. Базовые технические команды

## Backend

```bash
php artisan test
composer test
php artisan route:list
php artisan schedule:list
php artisan migrate --pretend
```

Примечание: `composer test` сейчас выполняет `php artisan config:clear --ansi` и затем `php artisan test` согласно `composer.json` backend.

## Frontend user app

```bash
npm run build
```

Для backend-side Vite/Tailwind assets команда также актуальна из `aniyume-backend`, потому что в backend есть `package.json` и Blade-admin assets.

## Дополнительно по changed PHP files

```bash
php -l path/to/file.php
```

---

## 3. Minimum critical regression set

Это минимальный набор, который должен проходить всегда.

### 3.1. Auth
- [ ] Регистрация пользователя работает
- [ ] Логин пользователя работает
- [ ] Возвращается token / auth state
- [ ] `GET /api/v1/user` возвращает текущего пользователя
- [ ] Logout работает

### 3.2. Public catalog
- [ ] `GET /api/v1/public/anime` работает
- [ ] Поиск по anime работает
- [ ] Фильтрация по genre/type/status работает
- [ ] Детальная страница anime открывается
- [ ] Эпизоды anime возвращаются
- [ ] Player sources endpoint работает
- [ ] Tags endpoint работает

### 3.3. User content
- [ ] `POST /api/v1/anime/{anime}/status` добавляет/обновляет anime в списке
- [ ] `GET /api/v1/anime/{anime}/user-status` возвращает текущий статус
- [ ] `PATCH /api/v1/anime/{anime}/episodes-watched/{episodesWatched}` обновляет episodes watched
- [ ] `POST /api/v1/favorites` добавляет в favorites
- [ ] `DELETE /api/v1/favorites/{animeId}` удаляет из favorites
- [ ] `POST /api/v1/ratings` отправляет рейтинг
- [ ] `DELETE /api/v1/ratings/{rating}` удаляет рейтинг
- [ ] `POST /api/v1/comments` добавляет комментарий
- [ ] `PUT/PATCH /api/v1/comments/{comment}` редактирует комментарий
- [ ] `DELETE /api/v1/comments/{comment}` удаляет комментарий

### 3.4. Watch history
- [ ] Сохранение прогресса просмотра работает
- [ ] История просмотра возвращается
- [ ] Последний просмотренный эпизод возвращается
- [ ] Удаление записи истории работает

### 3.5. Social
- [ ] Поиск пользователей работает
- [ ] Отправка friend request работает
- [ ] Принятие заявки в друзья работает
- [ ] Список друзей возвращается
- [ ] Friendship status endpoint работает

### 3.6. Watch Party
- [ ] Создание комнаты работает
- [ ] Просмотр комнаты работает
- [ ] Join room работает
- [ ] Leave room работает
- [ ] Sync state работает
- [ ] Отправка сообщения работает
- [ ] История сообщений работает
- [ ] Invite friend работает
- [ ] Close room работает

### 3.7. Frontend build
- [ ] пользовательский frontend build проходит успешно, если изменения затрагивают frontend
- [ ] `aniyume-backend` `npm run build` проходит успешно, если изменения затрагивают Blade/admin assets

---

## 4. Extended backend regression set

Перед ручной проверкой endpoint names сверять с `backend-endpoint-inventory.md` и `php artisan route:list`.

## Public API
- [ ] `/api/v1/public/schedule` работает
- [ ] `/api/v1/public/anime/{anime}/banner` работает
- [ ] `/api/v1/public/anime/{anime}/community-stats` работает
- [ ] `/api/v1/public/anime/{anime}/recommendations` работает
- [ ] `/api/v1/public/episodes` работает
- [ ] `/api/v1/public/episodes/{episode}` работает
- [ ] `/api/v1/public/episodes/{episode}/player` работает

## Profile/statistics
- [ ] Профиль пользователя возвращается
- [ ] Обновление профиля работает
- [ ] Upload avatar работает
- [ ] Статистика пользователя возвращается
- [ ] Episodes summary работает

## Data consistency
- [ ] После комментария корректно отражается список комментариев
- [ ] После рейтинга корректно отражается пользовательская оценка
- [ ] После обновления статуса anime корректно отражается user status
- [ ] После watch history корректно отражается last watched episode

---

## 5. Admin regression set

Пока Blade-admin еще существует, этот набор обязателен.

### Admin auth
- [ ] `/admin/login` открывается
- [ ] Admin login работает
- [ ] Admin logout работает

### Admin dashboard
- [ ] Dashboard открывается
- [ ] Summary metrics отображаются
- [ ] Latest anime отображаются
- [ ] Recent imports отображаются

### Admin anime
- [ ] Список anime открывается
- [ ] Поиск anime работает
- [ ] Создание anime работает
- [ ] Редактирование anime работает
- [ ] Удаление anime работает
- [ ] Bulk delete работает

### Admin tags
- [ ] Список tags открывается
- [ ] Создание tag работает
- [ ] Редактирование tag работает
- [ ] Удаление tag работает

### Admin episodes
- [ ] Список episodes открывается
- [ ] Фильтр по anime работает
- [ ] Редактирование episode работает
- [ ] Import for anime запускается
- [ ] Bulk import запускается
- [ ] Delete episode работает

### Admin comments moderation
- [ ] Список comments moderation открывается
- [ ] Approve comment работает
- [ ] Reject comment работает
- [ ] Delete comment работает

### Admin imports
- [ ] Import dashboard открывается
- [ ] Run import работает
- [ ] Import logs открываются

### Admin audit logs
- [ ] Audit logs page открывается
- [ ] Filters по action/user/date работают

---

## 6. Realtime regression set

### Broadcast auth
- [ ] Broadcast auth проходит для авторизованного пользователя
- [ ] Private user channel работает
- [ ] Presence channel watch party работает

### Watch party realtime
- [ ] Второй участник видит room presence
- [ ] Sync play/pause/seek доходит до участников
- [ ] Chat message приходит realtime
- [ ] Room close event приходит участникам
- [ ] Invite event доставляется другу

---

## 7. Database and migration checks

- [ ] Миграции применяются без неожиданных ошибок
- [ ] `php artisan migrate --pretend` проходит
- [ ] Критичные сиды/роли существуют
- [ ] Admin role присутствует
- [ ] Test/demo users подготовлены

---

## 8. Deployability checks

### До Docker
- [ ] `.env` заполнен корректно
- [ ] storage link настроен
- [ ] queue worker запускается
- [ ] scheduler list корректен
- [ ] reverb config корректен

### После Docker
- [ ] `docker compose up --build` поднимает систему
- [ ] backend доступен
- [ ] frontend доступен
- [ ] admin доступен
- [ ] db доступна
- [ ] redis доступен
- [ ] queue worker жив
- [ ] scheduler service жив
- [ ] reverb сервис жив

---

## 9. Минимальный набор перед демонстрацией диплома

- [ ] frontend работает
- [ ] backend работает
- [ ] admin работает
- [ ] есть рабочий admin account
- [ ] есть минимум 2 user accounts для social/watch party demo
- [ ] queue worker запущен
- [ ] reverb запущен
- [ ] imports не падают
- [ ] public catalog работает
- [ ] login работает
- [ ] watch history работает
- [ ] comments and ratings работают
- [ ] watch party demo работает в 2 клиентах
- [ ] все docs актуальны

---

## 10. Рекомендуемый порядок проверки после больших изменений

### После изменений в auth
1. register
2. login
3. me
4. logout
5. protected endpoints

### После изменений в public catalog
1. public anime list
2. filters/search
3. anime details
4. episodes
5. player sources
6. recommendations

### После изменений в social/watch party
1. friends flow
2. create room
3. join room
4. chat
5. sync
6. invite
7. close room

### После изменений в admin
1. admin login
2. dashboard
3. anime CRUD
4. episodes import
5. comments moderation
6. import logs

---

## 11. Итог

Этот checklist — обязательная страховка от регрессий. Любой крупный refactor должен завершаться не ощущением «вроде работает», а подтверждением по конкретным сценариям.
