# Методические указания №2 FastAPI (SQLAlchemy, Alembic, PostgreSQL, Pydantic Settings)

Для выполнения лабораторной работы потребуется установленный Python 3.10+, пакетный менеджер pip, редактор кода (рекомендуется Visual Studio Code или PyCharm) и установленный Docker (для запуска PostgreSQL и Adminer). 

В данной лабораторной работе мы продолжим развитие проекта из первой части: откажемся от хранения данных в памяти (списков), настроим реальную реляционную базу данных PostgreSQL, внедрим систему миграций Alembic и реализуем мягкое удаление (soft delete), сохраняя при этом шаблонизацию через Jinja2.

## План работы
1. Работа с PostgreSQL (Запуск в Docker)
2. Подключение к БД через IDE
3. Конфигурация и переменные окружения (Pydantic Settings)
4. Подключение к БД (SQLAlchemy + Alembic)
5. Перенос данных в PostgreSQL и обновление модели
6. Обновление логики и страница «подробнее» (использование "курсора" / Raw SQL)
7. Удаление услуги (мягкое удаление) и обновление CSS
8. Итоговая структура проекта
9. FAQ

---

### 1. Работа с PostgreSQL
PostgreSQL — это мощная реляционная СУБД. На этом этапе мы поднимем её и веб-инструмент Adminer в контейнерах, чтобы иметь изолированную и воспроизводимую среду.

Создайте файл `docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: hotels_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    depends_on:
      - postgres

volumes:
  postgres_data:
```
Запуск выполняется командой:
```bash
docker-compose up -d
```
Проверить статус контейнеров можно командой `docker ps` или в приложении. Убедитесь, что контейнеры `postgres` и `adminer` находятся в состоянии «Running».

![docker](./assets/image-5.png)

---

### 2. Подключение к БД через IDE
Перед написанием кода полезно убедиться, что база данных доступна. Откройте pgAdmin и создайте новое соединение:
- **Host:** `localhost`
- **Port:** `5432`
- **Database:** `hotels_db`
- **Username:** `myuser`
- **Password:** `mypassword`

Нажмите «Save». Если БД успешно создана, то в pgadmin появится новое соединение.

![pgadmin](./assets/image-1.png)

---

### 3. Конфигурация и переменные окружения
В реальных проектах параметры подключения не хранят в коде. Мы будем использовать файл `.env` и библиотеку `pydantic-settings` для типизированного чтения настроек.

Установите необходимые пакеты:
```bash
pip install pydantic-settings python-dotenv asyncpg
pip freeze > requirements.txt
```

Создайте в корне проекта файл `.env`:
```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=hotels_db
```
*Важно: Добавьте `.env` в файл `.gitignore`.*

Создайте файл `core/config.py` для чтения настроек:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DB_HOST: str
    DB_PORT: int
    DB_USER: str
    DB_PASSWORD: str
    DB_NAME: str

    @property
    def DATABASE_URL(self) -> str:
        return f"postgresql+asyncpg://{self.DB_USER}:{self.DB_PASSWORD}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"

    class Config:
        env_file = ".env"

settings = Settings()
```

---

### 4. Подключение к БД (SQLAlchemy и Alembic)
SQLAlchemy позволяет работать с БД через объекты Python. Alembic управляет версиями схемы базы данных (миграциями).

Установите зависимости (если ещё не установили их):
```bash
pip install sqlalchemy alembic
pip freeze > requirements.txt
```

Создайте файл `db/base.py` (основа для моделей):
```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

Создайте файл `db/session.py` (движок и сессии):
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from core.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=False)
async_session_maker = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with async_session_maker() as session:
        yield session
```

Инициализируйте Alembic в корне проекта:
```bash
alembic init alembic
```
Откройте `alembic.ini` и измените строку `sqlalchemy.url` на заглушку (мы будем передавать URL из кода):
```ini
sqlalchemy.url = driver://user:pass@localhost/dbname
```
Откройте `alembic/env.py`, импортируйте настройки и модель, и укажите `target_metadata`:
```python
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

from core.config import settings
from db.base import Base
# Импортируйте здесь вашу модель, чтобы Alembic её увидел!
from models.hotel import Hotel 

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True, dialect_opts={"paramstyle": "named"})
    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

### 5. Перенос данных в PostgreSQL и обновление модели
Опишем сущность `Hotel`. Мы добавим поле `is_deleted` для реализации мягкого удаления.

Создайте файл `models/hotel.py`:
```python
from sqlalchemy import Column, Integer, String, Float, Boolean
from db.base import Base

class Hotel(Base):
    __tablename__ = "hotels"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(100), nullable=False)
    price = Column(Float, nullable=False)
    description = Column(String(500), nullable=False)
    image_url = Column(String(255), nullable=False)
    video_url = Column(String(255), nullable=False)
    is_deleted = Column(Boolean, default=False) # Поле для мягкого удаления
```

Создайте и примените миграцию:
```bash
alembic revision --autogenerate -m "init hotels table with soft delete"
alembic upgrade head
```

**Заполнение БД:**

Откройте Adminer (`http://localhost:8081`, система: PostgreSQL, сервер: postgres, пользователь: myuser, пароль: mypassword, БД: hotels_db). Перейдите в "SQL-команда" и выполните:
```sql
INSERT INTO hotels (id, title, price, description, image_url, video_url, is_deleted)
VALUES
  (1, 'Отель Морской бриз', 5000.0, 'Прекрасный отель на берегу моря.', 'http://localhost:9000/media/hotel.jpg', 'http://localhost:9000/media/hotel_vid.mp4', false),
  (2, 'Отель Горная вершина', 3500.0, 'Уютное шале в горах.', 'http://localhost:9000/media/mountains.jpg', 'http://localhost:9000/media/mountains.mp4', false);
```

![adminer добавление](./assets/image-2.png)

![adminer таблица](./assets/image-3.png)

---

### 6. Обновление логики и Страница «подробнее» (Курсор)
Теперь заменим данные из `data/collections.py` на запросы к БД. Особое внимание уделим методу получения отеля по ID, который, согласно нашему заданию, должен использовать курсор (выполнение сырого SQL-запроса и построчное чтение результата).

Обновите файл `api/handlers.py`:
```python
from fastapi import APIRouter, Request, Depends, HTTPException
from fastapi.templating import Jinja2Templates
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession
from db.session import get_db

router = APIRouter()
templates = Jinja2Templates(directory="templates")

@router.get("/")
async def get_catalog(request: Request, search: str = "", db: AsyncSession = Depends(get_db)):
    # Формируем запрос с учетом мягкого удаления
    query = "SELECT id, title, price, description, image_url, video_url FROM hotels WHERE is_deleted = false"
    params = {}
    
    if search:
        query += " AND title ILIKE :search"
        params["search"] = f"%{search}%"
        
    result = await db.execute(text(query), params)
    hotels = result.mappings().all() # Возвращает список словареподобных объектов
    
    return templates.TemplateResponse(
        request=request,
        name="index.html", 
        context={"hotels": hotels, "search": search}
    )

@router.get("/hotel/{hotel_id}")
async def get_hotel_detail(request: Request, hotel_id: int, db: AsyncSession = Depends(get_db)):
    # РЕАЛИЗАЦИЯ ЧЕРЕЗ "КУРСОР" (Raw SQL)
    # Мы явно выполняем SQL-запрос и читаем одну строку результата
    raw_query = """
        SELECT id, title, price, description, image_url, video_url 
        FROM hotels 
        WHERE id = :id AND is_deleted = false
    """
    
    # Выполняем запрос
    cursor_result = await db.execute(text(raw_query), {"id": hotel_id})
    
    # "Сканируем" первую строку (аналог cursor.fetchone())
    row = cursor_result.fetchone()
    
    if not row:
        raise HTTPException(status_code=404, detail="Отель не найден или удален")
        
    # Преобразуем строку в словарь для передачи в шаблон
    hotel = dict(row._mapping)
    
    return templates.TemplateResponse(
        request=request,
        name="hotel.html", 
        context={"hotel": hotel}
    )
```

---

### 7. Удаление услуги (Мягкое удаление) и обновление CSS
Мягкое удаление (Soft Delete) означает, что мы не удаляем запись из БД командой `DELETE`, а меняем флаг `is_deleted` на `true`. Такие записи перестают отображаться в списке, но сохраняются для истории и аналитики.

Добавьте обработчик удаления в `api/handlers.py`:
```python
from fastapi import Response

@router.post("/hotel/{hotel_id}/delete")
async def delete_hotel(hotel_id: int, db: AsyncSession = Depends(get_db)):
    # Мягкое удаление: обновляем флаг is_deleted
    update_query = """
        UPDATE hotels 
        SET is_deleted = true 
        WHERE id = :id
    """
    await db.execute(text(update_query), {"id": hotel_id})
    await db.commit()
    
    # Перенаправляем пользователя обратно на главную страницу
    return Response(status_code=303, headers={"Location": "/"})
```

**Обновление стилей (CSS):**

Добавьте следующие стили в ваш файл `static/css/style.css`:

```css
/* Стили для поисковой формы */
.search-form {
    display: flex;
    gap: 10px;
    margin-bottom: 30px;
    max-width: 600px;
}

.search-form input[type="text"] {
    flex: 1;
    padding: 10px 15px;
    border: 2px solid #005df9; /* Электрический синий */
    border-radius: 8px;
    font-size: 16px;
    outline: none;
    background-color: #ffffff;
    color: #333;
}

.search-form input[type="text"]::placeholder {
    color: #888;
}

.search-form button {
    padding: 10px 20px;
    background-color: #005df9;
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
    transition: background-color 0.2s ease;
}

.search-form button:hover {
    background-color: #0046b8;
}

/* Стили для кнопки удаления */
.delete-btn {
    background-color: #ff4444;
    color: white;
    border: none;
    padding: 8px 12px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.2s ease;
}

.delete-btn:hover {
    background-color: #cc0000;
}
```

Обновите шаблон `templates/index.html`, применив новые классы и добавив форму удаления в цикл вывода:
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Каталог отелей</title>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <h1>Каталог отелей</h1>
    
    <!-- Обновленная форма поиска с классами -->
    <form class="search-form" action="/" method="GET">
        <input type="text" name="search" placeholder="Поиск по названию" value="{{ search }}">
        <button type="submit">Найти</button>
    </form>

    <ul>
        {% for hotel in hotels %}
        <li>
            <a href="/hotel/{{ hotel.id }}">
                <img src="{{ hotel.image_url }}" alt="{{ hotel.title }}" class="hotel-image">            
                <h2>{{ hotel.title }}</h2>
                <p>Цена: {{ hotel.price }} руб.</p>
            </a>
            
            <!-- Форма мягкого удаления -->
            <form action="/hotel/{{ hotel.id }}/delete" method="POST" style="margin-top: 15px;">
                <button type="submit" class="delete-btn">
                    Удалить
                </button>
            </form>
        </li>
        {% endfor %}
    </ul>
</body>
</html>
```
*Примечание: Убедитесь, что в `main.py` добавлен `app.mount("/static", StaticFiles(directory="static"), name="static")`, как это было сделано в первой лабораторной.*

![итоговый вид проекта](./assets/image-4.png)

---

### 8. Итоговая структура проекта
После выполнения всех шагов структура вашего проекта должна выглядеть следующим образом:

```text
project_root/
├── alembic/
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
├── alembic.ini
├── api/
│   └── handlers.py
├── core/
│   └── config.py
├── db/
│   ├── base.py
│   └── session.py
├── models/
│   └── hotel.py
├── static/
│   ├── css/
│   │   └── style.css
├── templates/
│   ├── index.html
│   └── hotel.html
├── .env
├── .gitignore
├── main.py
└── requirements.txt
```

---

### 9. FAQ

**1. Почему мы используем `asyncpg` вместо стандартного `psycopg2`?**  
FastAPI построен на асинхронной архитектуре (ASGI). Использование асинхронного драйвера `asyncpg` позволяет базе данных не блокировать поток выполнения при ожидании ответа, что критически важно для высокой производительности и конкурентной обработки запросов.

**2. Зачем нужен Alembic, если есть параметр `synchronize` или создание таблиц при старте?**  
Автоматическая синхронизация (`synchronize=True` или `create_all`) опасна в продакшене: она может случайно удалить данные или некорректно изменить типы колонок. Alembic позволяет контролировать изменения схемы БД шаг за шагом, хранить историю изменений (версионность) и безопасно применять их на разных окружениях.

**3. В чем преимущество "курсора" (Raw SQL) перед стандартным ORM-методом `get()`?**  
Использование сырого SQL через `text()` и построчное чтение (`fetchone`) дает полный контроль над запросом. Это полезно для сложных выборок, оптимизации производительности (когда ORM генерирует неэффективные запросы с лишними JOIN) или при работе с унаследованными базами данных. В учебных целях это демонстрирует понимание того, как фреймворк взаимодействует с СУБД "под капотом".

**4. Почему удаление называется "мягким" (Soft Delete)?**  
При мягком удалении запись физически остается в таблице, но меняется её статус (`is_deleted = true`). Это позволяет сохранить целостность внешних ключей (например, если в будущем вы добавите связанные сущности, вроде бронирований, они не потеряют связь с отелем), вести аудит действий пользователей и при необходимости восстановить удаленные данные.