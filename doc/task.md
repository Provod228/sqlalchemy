# Документация проекта "Демонстрационный стенд SQLAlchemy"

## Структура проекта

```
sqlalchemy-demo/
├── library_demo.py    # Модели данных, ORM-операции и демонстрационные сценарии
├── library.db         # SQLite база данных (создается автоматически)
└── README.md          # Документация проекта
```

## Описание модулей

### `library_demo.py` — основной модуль системы

Содержит модели данных для библиотечной системы, ORM-операции, демонстрационные сценарии и примеры оптимизации запросов.

**Модели данных:**
- **`Author`** — автор книг (`id`, `name`, `email`, `books`)
- **`Book`** — книга (`id`, `title`, `price`, `author_id`, `author`)

**Демонстрационные сценарии:**
- Создание данных с использованием связей (каскадное сохранение)
- Базовые CRUD-запросы (SELECT, фильтрация, сортировка)
- Работа со связями и демонстрация проблемы N+1
- Агрегатные функции и группировка данных
- Обновление и каскадное удаление записей
- Сложные запросы с подзапросами

## Как использовать

### Установка и запуск

1. Убедитесь, что у вас установлен Python 3.8 или выше

2. Установите SQLAlchemy:
```bash
pip install sqlalchemy
```

3. Скачайте файл `library_demo.py`

4. Запустите демонстрацию:
```bash
python library_demo.py
```

### Работа с приложением

При запуске скрипт последовательно выполняет шесть демонстрационных блоков:

1. **Создание данных** — демонстрация создания авторов и книг со связями
2. **Базовые запросы** — SELECT, фильтрация, сортировка
3. **Работа со связями** — сравнение lazy loading и eager loading (N+1 проблема)
4. **Агрегация и группировка** — статистические функции и GROUP BY
5. **Обновление и удаление** — UPDATE, DELETE и каскадное удаление
6. **Сложные запросы** — подзапросы и JOIN с условиями

Каждый блок сопровождается подробным выводом в консоль и замером времени выполнения.

## Модели данных

### 1. Author — автор книг

| Поле | Тип | Валидация | Описание |
|------|-----|-----------|----------|
| id | Integer | primary_key=True | Уникальный идентификатор автора |
| name | String(50) | nullable=False | Полное имя автора |
| email | String(100) | unique=True | Email адрес автора |
| books | relationship | back_populates="author", cascade="all, delete-orphan" | Список книг автора |

**Связи:**
- Один ко многим с моделью `Book`
- Каскадное удаление: при удалении автора удаляются все его книги

**Пример:**
```python
author = Author(
    name="Лев Толстой",
    email="tolstoy@example.com"
)
```

### 2. Book — книга

| Поле | Тип | Валидация | Описание |
|------|-----|-----------|----------|
| id | Integer | primary_key=True | Уникальный идентификатор книги |
| title | String(100) | nullable=False | Название книги |
| price | Float | default=0.0 | Цена книги |
| author_id | Integer | ForeignKey('authors.id', ondelete='CASCADE') | Внешний ключ на автора |
| author | relationship | back_populates="books" | Ссылка на объект автора |

**Связи:**
- Многие к одному с моделью `Author`

**Пример:**
```python
book = Book(
    title="Война и мир",
    price=599.0,
    author_id=1
)
```

## Демонстрационные сценарии

### 1. Создание данных (`demo_create_data`)

Демонстрирует создание связанных объектов с использованием relationship.

**Действия:**
- Создание двух авторов (Лев Толстой, Федор Достоевский)
- Добавление книг через свойство `books` (автоматически устанавливается author_id)
- Каскадное сохранение через `session.add_all()`

**Ключевые концепции:**
```python
# Связывание книг с автором через relationship
author1.books = [
    Book(title="Война и мир", price=599.0),
    Book(title="Анна Каренина", price=450.0)
]
# При добавлении автора книги сохранятся автоматически
session.add(author1)
```

### 2. Базовые запросы (`demo_query_basic`)

Демонстрирует основные операции SELECT.

**Типы запросов:**
- `session.query(Author).all()` — получение всех записей
- `session.query(Author).filter(Author.name == "Лев Толстой").first()` — фильтрация
- `session.query(Book).order_by(Book.price.desc()).limit(3).all()` — сортировка с ограничением

**Генерируемый SQL:**
```sql
-- Получение всех авторов
SELECT * FROM authors

-- Фильтрация по имени
SELECT * FROM authors WHERE name = 'Лев Толстой' LIMIT 1

-- Топ-3 дорогих книг
SELECT * FROM books ORDER BY price DESC LIMIT 3
```

### 3. Работа со связями (`demo_relationships`)

Демонстрирует проблему N+1 запросов и способы её решения.

**Проблема N+1 (Lazy Loading):**
```python
# 1 запрос для получения авторов
authors = session.query(Author).all()

# N дополнительных запросов для получения книг каждого автора
for author in authors:
    books_count = len(author.books)  # Каждый раз отдельный SELECT!
```

**Решение (Eager Loading):**
```python
# 1 запрос с подзапросом для загрузки всех данных
authors = session.query(Author).options(
    selectinload(Author.books)
).all()

# Нет дополнительных запросов
for author in authors:
    books_count = len(author.books)  # Данные уже загружены
```

**Сравнение подходов:**
| Подход | Количество запросов | Когда использовать |
|--------|--------------------|--------------------|
| Lazy Loading | 1 + N | Для небольших объемов данных |
| Eager Loading (selectinload) | 2 | Для предотвращения N+1 |
| Eager Loading (joinedload) | 1 (JOIN) | Когда нужны данные из обеих таблиц |

### 4. Агрегация и группировка (`demo_aggregation`)

Демонстрирует использование агрегатных функций SQL.

**Агрегатные функции:**
- `func.count()` — подсчет количества записей
- `func.avg()` — среднее арифметическое
- `func.min()` / `func.max()` — минимальное/максимальное значение
- `func.sum()` — сумма значений

**Примеры запросов:**
```python
# Общая статистика по всем книгам
stats = session.query(
    func.count(Book.id).label('total_books'),
    func.avg(Book.price).label('avg_price'),
    func.min(Book.price).label('min_price'),
    func.max(Book.price).label('max_price'),
    func.sum(Book.price).label('total_value')
).first()

# Группировка по авторам
author_stats = session.query(
    Author.name,
    func.count(Book.id).label('book_count'),
    func.avg(Book.price).label('avg_price')
).join(Book).group_by(Author.id).all()
```

**Генерируемый SQL:**
```sql
-- Общая статистика
SELECT 
    COUNT(books.id) AS total_books,
    AVG(books.price) AS avg_price,
    MIN(books.price) AS min_price,
    MAX(books.price) AS max_price,
    SUM(books.price) AS total_value
FROM books

-- Группировка по авторам
SELECT 
    authors.name,
    COUNT(books.id) AS book_count,
    AVG(books.price) AS avg_price
FROM authors
JOIN books ON authors.id = books.author_id
GROUP BY authors.id
```

### 5. Обновление и удаление (`demo_update_delete`)

Демонстрирует операции UPDATE и DELETE с каскадным удалением.

**UPDATE:**
```python
# Получение объекта
tolstoy = session.query(Author).filter(Author.name == "Лев Толстой").first()

# Изменение атрибутов связанных объектов
for book in tolstoy.books:
    book.price = round(book.price * 1.1, 2)  # Увеличение цены на 10%

# Сохранение изменений
session.commit()
```

**DELETE с каскадом:**
```python
# Удаление автора
dostoevsky = session.query(Author).filter(Author.name == "Федор Достоевский").first()
session.delete(dostoevsky)
session.commit()
# Все книги автора удаляются автоматически благодаря cascade="all, delete-orphan"
```

**Каскадные опции:**
- `cascade="all"` — все операции передаются связанным объектам
- `cascade="delete-orphan"` — удаление объектов, потерявших связь
- `ondelete='CASCADE'` на уровне БД (внешний ключ)

### 6. Сложный запрос с подзапросом (`demo_complex_query`)

Демонстрирует создание сложных запросов с подзапросами.

**Задача:** Найти книги, цена которых выше средней цены книг этого же автора.

**Решение:**
```python
# Подзапрос: средняя цена для каждого автора
subquery = session.query(
    Book.author_id,
    func.avg(Book.price).label('avg_author_price')
).group_by(Book.author_id).subquery()

# Основной запрос с JOIN подзапроса
expensive_books = session.query(
    Author.name.label('author'),
    Book.title,
    Book.price,
    subquery.c.avg_author_price
).join(Book).join(
    subquery, Book.author_id == subquery.c.author_id
).filter(
    Book.price > subquery.c.avg_author_price
).all()
```

**Генерируемый SQL:**
```sql
SELECT 
    authors.name AS author,
    books.title,
    books.price,
    subq.avg_author_price
FROM books
JOIN authors ON books.author_id = authors.id
JOIN (
    SELECT 
        author_id,
        AVG(price) AS avg_author_price
    FROM books
    GROUP BY author_id
) AS subq ON books.author_id = subq.author_id
WHERE books.price > subq.avg_author_price
```

## Ключевые концепции SQLAlchemy

### 1. Движок (Engine) и подключение к БД

```python
# SQLite (файловая БД)
engine = create_engine('sqlite:///library.db')

# PostgreSQL
engine = create_engine('postgresql://user:pass@localhost/db')

# MySQL
engine = create_engine('mysql://user:pass@localhost/db')
```

### 2. Декларативные модели

```python
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class MyModel(Base):
    __tablename__ = 'table_name'  # Имя таблицы в БД
    id = Column(Integer, primary_key=True)  # Колонка
```

### 3. Типы колонок

| Тип | Описание |
|-----|----------|
| Integer | Целое число |
| String(length) | Строка фиксированной/переменной длины |
| Float | Число с плавающей точкой |
| DateTime | Дата и время |
| Boolean | Логическое значение |
| ForeignKey | Внешний ключ |

### 4. Отношения (Relationships)

```python
# Один ко многим
class Parent(Base):
    children = relationship("Child", back_populates="parent")

class Child(Base):
    parent_id = Column(Integer, ForeignKey('parents.id'))
    parent = relationship("Parent", back_populates="children")
```

### 5. Сессии (Session)

```python
from sqlalchemy.orm import Session

session = Session(engine)

# Операции с БД
session.add(object)      # Добавление объекта
session.add_all([obj1, obj2])  # Добавление нескольких
session.commit()          # Фиксация изменений
session.rollback()        # Откат изменений
session.close()           # Закрытие сессии
```

### 6. Запросы (Query API)

```python
# Базовые запросы
session.query(Model).all()                    # Все записи
session.query(Model).first()                  # Первая запись
session.query(Model).get(id)                  # По первичному ключу

# Фильтрация
session.query(Model).filter(Model.field == value)
session.query(Model).filter_by(field=value)   # Упрощенная форма

# Сортировка
session.query(Model).order_by(Model.field.desc())
session.query(Model).order_by(Model.field.asc())

# Лимит и смещение (пагинация)
session.query(Model).limit(10).offset(20)

# Агрегация
session.query(func.count(Model.id)).scalar()
```

### 7. Жадная загрузка (Eager Loading)

```python
from sqlalchemy.orm import selectinload, joinedload

# selectinload - второй запрос с IN (рекомендуется)
query = session.query(Parent).options(selectinload(Parent.children))

# joinedload - один запрос с LEFT JOIN
query = session.query(Parent).options(joinedload(Parent.children))
```

### 8. Контекстный менеджер для сессий

```python
from contextlib import contextmanager

@contextmanager
def session_scope():
    session = Session(engine)
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Использование
with session_scope() as session:
    author = Author(name="New Author")
    session.add(author)
# Автоматический commit или rollback + закрытие
```

## Примеры вывода

### Вывод демонстрации создания данных:
```
==================================================
📚 СОЗДАНИЕ ДАННЫХ
==================================================
✅ Добавлено авторов: 2
✅ Добавлено книг: 6
⏱ Время выполнения: 0.0123 сек
```

### Вывод демонстрации N+1 проблемы:
```
==================================================
📚 РАБОТА СО СВЯЗЯМИ
==================================================

🐌 LAZY LOADING (N+1 запросов):
  Лев Толстой: 3 книг
  Федор Достоевский: 3 книг

⚡ EAGER LOADING (1 запрос):
  Лев Толстой: 3 книг
  Федор Достоевский: 3 книг
⏱ Время выполнения: 0.0087 сек
```

### Вывод агрегации:
```
==================================================
📚 АГРЕГАЦИЯ И ГРУППИРОВКА
==================================================

📊 Статистика по библиотеке:
  Всего книг: 6
  Средняя цена: 521.33 руб.
  Минимальная цена: 399.0 руб.
  Максимальная цена: 650.0 руб.
  Общая стоимость: 3128.0 руб.

📖 Книги по авторам:
  Лев Толстой: 3 книг, средняя цена 483 руб.
  Федор Достоевский: 3 книг, средняя цена 560 руб.
```

## Оптимизация производительности

### Сравнение подходов загрузки данных

| Подход | Запросы | Время (100 авторов × 10 книг) |
|--------|---------|-------------------------------|
| Lazy Loading | 1 + 100 | ~0.5 сек |
| selectinload | 2 | ~0.05 сек |
| joinedload | 1 | ~0.04 сек |

### Рекомендации по оптимизации

1. **Используйте selectinload для one-to-many связей**
   ```python
   .options(selectinload(Author.books))
   ```

2. **Используйте joinedload для many-to-one связей**
   ```python
   .options(joinedload(Book.author))
   ```

3. **Загружайте только нужные поля**
   ```python
   session.query(Book.id, Book.title).all()
   ```

4. **Используйте пагинацию для больших наборов данных**
   ```python
   session.query(Book).limit(100).offset(200)
   ```

5. **Применяйте индексы на часто фильтруемых полях**
   ```python
   from sqlalchemy import Index
   
   class Book(Base):
       __tablename__ = 'books'
       __table_args__ = (Index('idx_price', 'price'),)
   ```

## Возможные ошибки и их решение

### 1. DetachedInstanceError

**Проблема:** Доступ к lazy-loaded атрибуту после закрытия сессии
```python
session.close()
print(author.books)  # Ошибка!
```

**Решение:** Загрузить данные до закрытия сессии
```python
# Eager loading
author = session.query(Author).options(selectinload(Author.books)).first()
session.close()
print(author.books)  # Работает!
```

### 2. IntegrityError

**Проблема:** Нарушение уникальности или NOT NULL constraint
```python
session.add(Author(name="John", email="test@example.com"))
session.add(Author(name="John", email="test@example.com"))  # Ошибка!
session.commit()
```

**Решение:** Проверка перед добавлением
```python
exists = session.query(Author).filter_by(email="test@example.com").first()
if not exists:
    session.add(Author(name="John", email="test@example.com"))
    session.commit()
```

### 3. N+1 запросов

**Проблема:** Неявные дополнительные запросы в цикле
```python
for author in session.query(Author).all():
    print(len(author.books))  # Дополнительный запрос для каждого!
```

**Решение:** Использование eager loading
```python
for author in session.query(Author).options(selectinload(Author.books)).all():
    print(len(author.books))  # Нет дополнительных запросов
```

## Заключение

SQLAlchemy предоставляет мощный и гибкий инструментарий для работы с реляционными базами данных в Python. Данный демонстрационный стенд наглядно показывает:

- **Абстракцию** — работа с объектами Python вместо SQL строк
- **Производительность** — оптимизация запросов через eager loading
- **Безопасность** — защита от SQL-инъекций через параметризованные запросы
- **Поддерживаемость** — самодокументируемые модели данных

Проект может служить отправной точкой для изучения SQLAlchemy и использования ORM в реальных проектах.
