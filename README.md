# Спортивные площадки Москвы — план от 0 до рабочего проекта

Сайт-справочник: какая площадка, для чего, когда открыта. Frontend — React, backend — FastAPI, данные — data.mos.ru + OpenStreetMap.

## Быстрый чеклист по этапам

- [ ] Этап 0 — окружение и структура репозитория
- [ ] Этап 1 — сбор и нормализация данных
- [ ] Этап 2 — база данных (PostgreSQL + PostGIS)
- [ ] Этап 3 — backend API (FastAPI)
- [ ] Этап 4 — frontend (React + карта)
- [ ] Этап 5 — полировка
- [ ] Этап 6 — деплой
- [ ] Этап 7 (опционально) — v2

Очень грубая оценка времени в режиме "по вечерам/выходным": этапы 0–2 — около недели, этап 3 — неделя–полторы, этап 4 — две недели, этапы 5–6 — неделя. Сильно зависит от опыта и того, сколько времени реально удаётся выделять.

---

## Этап 0. Окружение и структура репозитория

**Что установить:** Python 3.11+, Node.js 20+, PostgreSQL с расширением PostGIS (проще всего через Docker), Git. Docker — по желанию, но сильно упрощает жизнь с PostGIS.

```bash
mkdir sport-grounds && cd sport-grounds
mkdir backend frontend
git init
```

Структура репозитория, к которой стоит прийти к концу этапа 3-4:

```text
sport-grounds/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── database.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── admin.py
│   │   └── routers/
│   │       └── grounds.py
│   ├── alembic/
│   ├── scripts/
│   │   └── import_data.py
│   ├── requirements.txt
│   └── docker-compose.yml
└── frontend/
    ├── src/
    │   ├── components/
    │   │   ├── MapView.tsx
    │   │   ├── FilterPanel.tsx
    │   │   └── GroundCard.tsx
    │   ├── api/
    │   │   └── grounds.ts
    │   ├── App.tsx
    │   └── main.tsx
    ├── package.json
    └── vite.config.ts
```

Поднять Postgres с PostGIS локально (без ручной установки):

```bash
docker run --name pg-sport \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=sportgrounds \
  -p 5432:5432 -d postgis/postgis:16-3.4
```

**Готово, когда:** есть репозиторий с папками backend/frontend, поднят Postgres и вы можете подключиться к нему любым клиентом (psql, DBeaver, TablePlus).

---

## Этап 1. Данные

Самая нудная, но самая важная часть — от качества данных зависит всё остальное.

1. Зайти на портал открытых данных Москвы, найти датасет «Спортивные площадки» (`data.mos.ru`), скачать экспорт (обычно JSON/CSV/XML — формат уточнить на странице датасета).
2. Открыть файл и посмотреть на реальные поля: адрес, координаты, график работы, услуги, телефон, сайт организации. Не все поля будут заполнены у всех записей — это нормально для гос. открытых данных.
3. Опционально — дополнить через Overpass API (OpenStreetMap), если хочется покрыть частные/недавно построенные площадки:

```text
[out:json];
area["name"="Москва"]->.a;
(
  node["leisure"~"pitch|sports_centre|fitness_centre"](area.a);
  way["leisure"~"pitch|sports_centre|fitness_centre"](area.a);
);
out center;
```

4. Написать скрипт нормализации (`scripts/normalize.py`), который приводит все записи к единой схеме:
   - парсит график работы в структуру `{"mon": [["07:00","22:00"]], ...}`
   - помечает площадки без явного графика флагом `always_open = true` (типично для дворовых площадок)
   - помечает сезонные объекты флагом `seasonal` (катки, летние площадки)
   - разносит виды спорта по единому справочнику (футбол, баскетбол, воркаут, теннис, каток...)
   - убирает дубли по совпадению координат/адреса

**Результат этапа:** один чистый `grounds.json` (или CSV) с унифицированной схемой полей, готовый к загрузке в базу.

**Готово, когда:** у вас есть файл с 200+ площадками, где у каждой заполнены координаты, название и хотя бы один вид спорта.

---

## Этап 2. База данных

```bash
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install fastapi "uvicorn[standard]" sqlalchemy alembic psycopg2-binary geoalchemy2 pydantic-settings python-dotenv sqladmin
pip freeze > requirements.txt
```

1. Создать `app/database.py` — подключение к БД через SQLAlchemy (движок, сессия, `Base`).
2. Создать `app/models.py` — модели `SportGround` и `SportType` (см. схему из предыдущего обсуждения: id, name, sport_types, address, district, location (geography Point), hours (jsonb), always_open, seasonal, phone, org_name, source, external_id).
3. Инициализировать Alembic и сделать первую миграцию:

```bash
alembic init alembic
# прописать DATABASE_URL в alembic/env.py или через .env
alembic revision --autogenerate -m "init tables"
alembic upgrade head
```

4. Написать `scripts/import_data.py` — читает `grounds.json` из этапа 1 и загружает записи в БД (upsert по `external_id`, чтобы можно было перезапускать импорт без дублей).

**Готово, когда:** в таблице `sport_grounds` реально лежат ваши данные, и вы можете сделать `SELECT * FROM sport_grounds LIMIT 5;` и увидеть осмысленные строки.

---

## Этап 3. Backend API (FastAPI)

1. `app/main.py` — создать приложение, подключить CORS (разрешить origin фронтенда), подключить роутеры.
2. `app/schemas.py` — Pydantic-схемы для ответов (`GroundOut`, `GroundListItem`, `SportTypeOut`).
3. `app/routers/grounds.py` — реализовать эндпоинты:

```text
GET  /api/grounds?sport=football&district=...&open_now=true&lat=..&lng=..&radius_m=2000&page=1
GET  /api/grounds/{id}
GET  /api/sport-types
GET  /api/districts
```

   Фильтр "рядом" — через PostGIS `ST_DWithin`, фильтр "открыто сейчас" — сравнение текущего времени с полем `hours` (с учётом `always_open`/`seasonal`).

4. Подключить `sqladmin` (`app/admin.py`) — админ-панель для ручной правки ошибок в данных без написания своего CRUD UI.
5. Запустить и проверить:

```bash
uvicorn app.main:app --reload
```

Открыть `http://localhost:8000/docs` — там должна быть рабочая Swagger-документация со всеми эндпоинтами, через которую можно руками потыкать API.

**Готово, когда:** через `/docs` можно получить список площадок, отфильтровать по виду спорта и получить одну площадку по id.

---

## Этап 4. Frontend (React + карта)

```bash
cd ../frontend
npm create vite@latest . -- --template react-ts
npm install
npm install leaflet react-leaflet react-leaflet-cluster @tanstack/react-query zustand
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Создать `.env` с адресом API:

```text
VITE_API_URL=http://localhost:8000
```

Порядок реализации компонентов (от простого к сложному):

1. `api/grounds.ts` — функции запроса к бэкенду (fetch/axios) + хуки TanStack Query (`useGrounds`, `useGround`).
2. `components/MapView.tsx` — карта Leaflet, маркеры из списка площадок, попап с названием и видами спорта.
3. `components/GroundCard.tsx` — карточка площадки для списка/попапа: название, виды спорта, адрес, статус "открыто/закрыто сейчас".
4. `components/FilterPanel.tsx` — фильтры: вид спорта (мультиселект), округ, тумблер "открыто сейчас", поиск по названию/адресу. Состояние фильтров — в Zustand или `useReducer`, пробрасывается и в карту, и в список.
5. `App.tsx` — сборка всего вместе: сайдбар со списком слева, карта справа (или наоборот на мобильных).
6. Кнопка геолокации "рядом со мной" — через `navigator.geolocation.getCurrentPosition`, дальше запрос с `lat/lng/radius_m`.

```bash
npm run dev
```

**Готово, когда:** на странице видна карта с реальными метками из вашего API, клик по метке показывает карточку, фильтр по виду спорта реально сужает список меток.

---

## Этап 5. Полировка

- Кластеризация меток (`react-leaflet-cluster`), когда точек становится много и они наслаиваются друг на друга.
- Мобильная адаптивность — сайдбар со списком должен схлопываться в шторку/таб на маленьких экранах.
- Состояния загрузки и ошибок (skeleton/spinner, сообщение при недоступности API).
- Favicon, title, базовые og:meta-теги, если планируете кому-то скидывать ссылку.

**Готово, когда:** сайт нормально открывается и пользуется на телефоне, а не только на широком мониторе.

---

## Этап 6. Деплой

1. `backend/Dockerfile` — образ с FastAPI-приложением.
2. `backend/docker-compose.yml` — backend + postgres/postgis для локальной сборки в связке.
3. Запушить репозиторий на GitHub.
4. Backend + база — задеплоить на Render/Railway (или свой VPS): прописать `DATABASE_URL`, разрешённые CORS-origin'ы для прод-домена фронта.
5. Frontend — статическая сборка на Vercel/Netlify: прописать `VITE_API_URL`, указывающий на прод-адрес backend.
6. Прогнать импорт данных (этап 2) уже на прод-базе.

**Готово, когда:** сайт открывается по публичной ссылке у кого-то ещё, не у вас на localhost.

---

## Этап 7 (опционально). Что добавить дальше

- Отзывы/рейтинги площадок от пользователей.
- Фото площадок.
- Кнопка "сообщить об ошибке в данных" — простой community-фидбек в админку.
- Периодическое автообновление данных из data.mos.ru по расписанию (APScheduler/cron).

---

## Финальный чеклист MVP

- [ ] Датасет скачан, нормализован, дубли убраны
- [ ] БД поднята, миграции применены, данные импортированы
- [ ] API отдаёт список/деталь/фильтры/справочники, `/docs` работает
- [ ] Карта показывает реальные маркеры, клик — карточка с деталями
- [ ] Фильтры по виду спорта и "открыто сейчас" работают корректно
- [ ] Сайт адаптивен на мобильном
- [ ] Сайт задеплоен и доступен по публичной ссылке
