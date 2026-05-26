                              # Anime Recommendations — Design Spec

                              **Date:** 2026-03-25
                              **Status:** Approved

                              ---

                              ## Overview

                              Заменить текущую реализацию "Похожее (Топ)" в `AnimeSidebar` на умные рекомендации:
                              - **Связанное** — официально связанные аниме (сиквелы, приквелы, спин-оффы) через Shikimori API
                              - **Похожее** — аниме с максимальным числом совпадающих жанров/тегов из нашей БД

                              Два отдельных блока в существующем сайдбаре. Оба блока всегда заполнены.

                              ---

                              ## Backend

                              ### Новый endpoint

                              ```
                              GET /public/anime/{id}/recommendations
                              ```

                              Полный путь: `/api/v1/public/anime/{id}/recommendations`

                              **Маршрут:** добавить в `routes/api.php` в группу публичных маршрутов (рядом с другими `public/anime` маршрутами).

                              **Метод:** `AnimeController::getRecommendations(Anime $anime)`

                              ### Логика метода

                              ```
                              1. related = [], has_official_related = false

                              2. Если $anime->shikimori_id существует:
                                  a. Http::timeout(5)->withHeaders(['User-Agent' => 'Aniyume/1.0'])->get(
                                          "https://shikimori.one/api/animes/{$shikimori_id}/related"
                                      )
                                  b. Если ответ успешен (2xx):
                                      - Из ответа взять объекты где anime != null
                                      - Собрать shikimori_id (как int) из этих объектов
                                      - Найти в нашей БД: Anime::whereIn('shikimori_id', [...ids])->limit(10)->get()
                                      - Смапить: добавить relation_type из ответа Shikimori к каждому найденному аниме
                                      - related = найденные аниме с relation_type
                                      - Если related непустой: has_official_related = true
                                  c. Если запрос упал (исключение / не 2xx): related = [], продолжаем

                              3. similar = Anime с наибольшим числом совпадающих тегов
                                - Исключить текущее аниме
                                - Исключить аниме из related (по id)
                                - Лимит: 5
                                - Алгоритм: JOIN anime_tag WHERE tag_id IN (теги текущего аниме),
                                            GROUP BY anime_id, ORDER BY COUNT(*) DESC LIMIT 5

                              4. Если related пустой (нет shikimori_id / API упал / нет совпадений в БД):
                                  related = первые 5 аниме по жанрам (те же запрос, что и similar)
                                  similar = следующие 5 (offset 5)
                                  has_official_related остаётся false

                              5. Вернуть { has_official_related, related, similar }
                              ```

                              ### Формат ответа

                              ```json
                              {
                                "has_official_related": true,
                                "related": [
                                  {
                                    "id": 12,
                                    "title": "Атака титанов: Финал",
                                    "poster_url": "https://...",
                                    "type": "tv",
                                    "rating": 9.1,
                                    "year": 2022,
                                    "relation_type": "Sequel"
                                  }
                                ],
                                "similar": [
                                  {
                                    "id": 34,
                                    "title": "Берсерк",
                                    "poster_url": "https://...",
                                    "type": "tv",
                                    "rating": 8.7,
                                    "year": 1997,
                                    "relation_type": null
                                  }
                                ]
                              }
                              ```

                              **Примечание:** используем `year` (не `release_year`) — соответствует полю в `AnimeResource` и типу `AnimeDetails` на фронте.

                              **Сериализация:** возвращать **минимальный кастомный массив** напрямую из метода (не через `AnimeResource`). Только нужные поля: `id`, `title`, `poster_url`, `type`, `rating`, `year`, `relation_type`. Пример для одного элемента:
                              ```php
                              [
                                  'id'            => $anime->id,
                                  'title'         => $anime->title,
                                  'poster_url'    => $anime->poster_url,
                                  'type'          => $anime->type,
                                  'rating'        => $anime->rating,
                                  'year'          => $anime->year,
                                  'relation_type' => $relationType ?? null,
                              ]
                              ```

                              ### HTTP клиент

                              Использовать Laravel HTTP Client (`Http` facade, Guzzle под капотом):
                              ```php
                              use Illuminate\Support\Facades\Http;

                              $response = Http::timeout(5)
                                  ->withHeaders(['User-Agent' => 'Aniyume/1.0'])  // Shikimori требует кастомный User-Agent
                                  ->get("https://shikimori.one/api/animes/{$anime->shikimori_id}/related");
                              ```

                              Весь вызов в `try/catch`: при любом исключении (таймаут, сетевая ошибка) — `related = []`, продолжаем.

                              ### phpDoc (для Scribe/Swagger)

                              ```php
                              /**
                              * Рекомендации для аниме
                              *
                              * Возвращает связанные (сиквелы/приквелы) и похожие по жанрам аниме.
                              *
                              * @group Публичные данные
                              * @urlParam anime integer ID аниме. Example: 3
                              * @response { "has_official_related": true, "related": [...], "similar": [...] }
                              */
                              ```

                              ---

                              ## Frontend

                              ### Изменения в `app/anime/[id]/page.tsx`

                              - Запрос за рекомендациями: заменить `/public/anime?sort=popularity&page=1` на `/public/anime/${id}/recommendations`
                              - Заменить state:
                                ```typescript
                                // было:
                                const [recommendations, setRecommendations] = useState<AnimeDetails[]>([]);

                                // стало:
                                const [related, setRelated] = useState<AnimeDetails[]>([]);
                                const [similar, setSimilar] = useState<AnimeDetails[]>([]);
                                const [hasOfficialRelated, setHasOfficialRelated] = useState(false);
                                ```
                              - Парсинг ответа:
                                ```typescript
                                setRelated(recJson.related || []);
                                setSimilar(recJson.similar || []);
                                setHasOfficialRelated(recJson.has_official_related ?? false);
                                ```
                              - Если fetch `/recommendations` упал — non-fatal: отдельный `try/catch` **только для этого запроса** (остальные три запроса остаются в `Promise.all` — ошибка там по-прежнему показывает error screen). При ошибке рекомендаций: `related = []`, `similar = []`, страница рендерится нормально.
                              - Поле `rating` в `AnimeDetails` типизировано как `string` (существующее поведение) — не менять при добавлении `relation_type`.
                              - Пробросить пропы в `<AnimeSidebar related={related} similar={similar} hasOfficialRelated={hasOfficialRelated} />`

                              ### Изменения в `components/watch/AnimeSidebar.tsx`

                              **Props:**
                              ```typescript
                              interface AnimeSidebarProps {
                                related: AnimeDetails[];
                                similar: AnimeDetails[];
                                hasOfficialRelated: boolean;
                              }
                              ```

                              **Структура:**

                              ```
                              Блок 1: "Связанное"
                                - Показывается только если related.length > 0
                                - Если hasOfficialRelated=true: показывать бейджи с типом связи
                                - Если hasOfficialRelated=false: бейджи не показывать (это жанровый фоллбэк)

                              Блок 2: "Похожее"
                                - Показывается только если similar.length > 0
                                - Если similar пустой: блок скрывается полностью (не показывать пустой UI)
                              ```

                              **Бейдж типа связи** (только при `hasOfficialRelated = true`):
                              - `text-xs`, `text-[#00E2C4]`, `bg-[#00E2C4]/10`, `rounded px-1.5 py-0.5`
                              - Показывается над названием аниме

                              **Функция `translateRelationType`:**
                              ```typescript
                              const RELATION_TYPES: Record<string, string> = {
                                'Sequel':        'Продолжение',
                                'Prequel':       'Приквел',
                                'Side Story':    'Побочная история',
                                'Alternative':   'Альтернативная версия',
                                'Summary':       'Краткое изложение',
                                'Adaptation':    'Адаптация',
                                'Spin-off':      'Спин-офф',
                                'Full Story':    'Полная версия',
                                'Parent Story':  'Основная история',
                                'Character':     'Персонаж',
                              };

                              function translateRelationType(type: string | null | undefined): string {
                                return RELATION_TYPES[type ?? ''] ?? 'Связанное';
                              }
                              ```

                              ### Изменения в `types/anime.ts`

                              Добавить к существующему типу `AnimeDetails`:
                              ```typescript
                              relation_type?: string | null;
                              ```

                              Это поле используется только в контексте рекомендаций, в остальных местах будет `undefined` — безопасно.

                              ---

                              ## Обработка ошибок

                              | Ситуация | Поведение |
                              |----------|-----------|
                              | Shikimori API недоступен / таймаут (5 сек) | `try/catch` → `related = []`, жанровый фоллбэк |
                              | Shikimori вернул не 2xx | То же — `related = []` |
                              | У аниме нет тегов | `similar = []`, блок "Похожее" скрывается полностью |
                              | Нет `shikimori_id` | Пропускаем вызов Shikimori, оба блока по жанрам |
                              | Fetch `/recommendations` упал на фронте | Non-fatal: `related = []`, `similar = []`, страница рендерится |
                              | Оба массива пустые | Сайдбар не показывает ни одного блока |

                              ---

                              ## Затронутые файлы

                              ### Backend
                              - `routes/api.php` — новый маршрут в группе `public`
                              - `app/Http/Controllers/Api/V1/AnimeController.php` — новый метод `getRecommendations`

                              ### Frontend
                              - `app/anime/[id]/page.tsx` — новый fetch, разделённый state, non-fatal обработка
                              - `components/watch/AnimeSidebar.tsx` — два блока, бейджи, пустые состояния
                              - `types/anime.ts` — добавить `relation_type?: string | null`

                              ---

                              ## Out of Scope

                              - Кэширование результатов Shikimori в БД (можно добавить позже)
                              - Персонализированные рекомендации на основе истории просмотров
                              - ML / embedding-based similarity
