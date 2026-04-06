# Документация проекта "Демонстрационный стенд SQLAlchemy"

## Модуль 1. Контент и предпосылки

### Ограничения стандартных средств + Описание проблем, которые существуют без данного инструмента

#### Проблема номер 1: Ручное написание SQL-запросов

Без ORM разработчикам приходится вручную писать SQL-запросы в виде строк. Это приводит к ошибкам, отсутствию проверки синтаксиса до выполнения и проблемам с поддержкой при изменении схемы БД.

```python
# Без SQLAlchemy - строки SQL в коде
def get_user_by_email(email):
    # Ошибка обнаружится только во время выполнения!
    cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
    # Проблема: SQL injection, нет проверки существования колонки
```

#### Проблема номер 2: Отсутствие типизации результатов запросов

Результаты запросов возвращаются в виде кортежей или словарей без информации о типах данных. Разработчик должен помнить порядок колонок и вручную преобразовывать типы.

```python
# Результат - кортеж без информации о типах
result = cursor.execute("SELECT id, name, age FROM users").fetchone()
user_id = result[0]  # Что это за тип? int? str?
user_name = result[1]  # А это точно строка?
```

#### Проблема номер 3: Сложность работы со связями между таблицами

Ручное управление внешними ключами и JOIN-запросами требует написания большого количества кода и приводит к дублированию логики.

```python
# Получение пользователя с его заказами без ORM
cursor.execute("""
    SELECT u.*, o.* 
    FROM users u 
    LEFT JOIN orders o ON u.id = o.user_id 
    WHERE u.id = ?
""", (user_id,))

# Затем нужно вручную группировать результаты
users_with_orders = {}
for row in cursor.fetchall():
    # Ручная группировка и создание объектов
    pass
```

#### Проблема номер 4: Отсутствие валидации на уровне БД в коде

Ограничения БД (NOT NULL, UNIQUE, FOREIGN KEY) проверяются только при выполнении запроса, что приводит к позднему обнаружению ошибок.

```python
# Ошибка обнаружится только при commit
cursor.execute("INSERT INTO users (email) VALUES (?)", (email,))
# Но поле name - NOT NULL! Ошибка только при commit
```

#### Проблема номер 5: Сложность миграций схемы БД

При изменении схемы БД нужно вручную писать скрипты миграции и обновлять все SQL-запросы в коде. Это трудоемко и подвержено ошибкам.

```python
# Добавили новую колонку в таблицу
# Нужно найти и обновить ВСЕ запросы, которые работают с этой таблицей
```

---

## Модуль 2. Основные идеи и механизмы SQLAlchemy

### 1. Центральные объекты и архитектура

Основным центральным объектом SQLAlchemy ORM является **Декларативная база (`declarative_base()`)**. Это фабрика, создающая базовый класс, от которого наследуются все модели. База хранит метаданные о всех определенных моделях.

Вторым ключевым объектом является **Движок (`Engine`)**. Это точка входа в базу данных, которая управляет пулом соединений и диалектами БД. Engine инкапсулирует информацию о подключении и стратегиях работы с конкретной СУБД.

Третий фундаментальный объект — **Сессия (`Session`)**. Это рабочая единица с БД, которая отслеживает изменения объектов, управляет транзакциями и выполняет запросы.

Архитектурно SQLAlchemy построена по принципу **"рабочих единиц" (Unit of Work)**. Сессия отслеживает все изменения в объектах и при коммите автоматически генерирует соответствующие SQL-запросы.

Важной частью архитектуры является **слой диалектов (`Dialect`)**. Он обеспечивает абстракцию от конкретной СУБД, преобразуя общий SQLAlchemy AST в диалект-специфичный SQL.

Еще один важный аспект — **система отношений (`relationship`)**. Она определяет, как связаны модели, и автоматически генерирует JOIN-запросы при доступе к связанным объектам.

Архитектура библиотеки также включает мощный слой **запросов (`Query API`)**. Он предоставляет цепочку методов для построения SQL-запросов без написания строк SQL.

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, Session, relationship

# Центральные объекты
Base = declarative_base()  # Базовый класс для моделей
engine = create_engine('sqlite:///demo.db')  # Движок для подключения

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    posts = relationship("Post")  # Отношение

class Post(Base):
    __tablename__ = 'posts'
    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey('users.id'))

# Сессия - рабочая единица
session = Session(engine)
```

---

### 2. Ключевые механизмы работы, возможности, принципы использования

#### 1) Механизм определения модели

Основной механизм. Вы создаете класс, наследующий от `Base`, и определяете поля как атрибуты класса `Column`.

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True)
```

#### 2) Механизм создания сессии

Сессия управляет транзакциями и отслеживает состояние объектов.

```python
from sqlalchemy.orm import Session

session = Session(engine)
user = User(name="Alice", email="alice@example.com")
session.add(user)
session.commit()  # Сохранение в БД
session.close()
```

#### 3) Механизм отношений (relationships)

Определение связей между моделями с автоматической загрузкой связанных данных.

```python
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = 'posts'
    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey('users.id'))
    author = relationship("User", back_populates="posts")
```

#### 4) Механизм запросов (Query API)

Построение SQL-запросов через цепочку методов.

```python
# SELECT * FROM users WHERE name = 'Alice'
users = session.query(User).filter(User.name == "Alice").all()

# SELECT * FROM users ORDER BY id DESC LIMIT 10
users = session.query(User).order_by(User.id.desc()).limit(10).all()

# SELECT COUNT(*) FROM users
count = session.query(User).count()
```

#### 5) Механизм жадной загрузки (Eager Loading)

Решение проблемы N+1 запросов через предварительную загрузку связанных данных.

```python
from sqlalchemy.orm import selectinload, joinedload

# Загрузка пользователей с их постами (2 запроса)
users = session.query(User).options(selectinload(User.posts)).all()

# Загрузка с JOIN (1 запрос, но возможны дубликаты)
users = session.query(User).options(joinedload(User.posts)).all()
```

#### 6) Механизм каскадных операций

Автоматическое выполнение операций над связанными объектами.

```python
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", cascade="all, delete-orphan")

# При удалении пользователя удалятся и его посты
session.delete(user)
session.commit()
```

#### 7) Механизм миграций (Alembic)

Интеграция с Alembic для управления изменениями схемы БД.

```python
# Создание миграции через командную строку
# alembic revision --autogenerate -m "add user table"
# alembic upgrade head
```

#### 8) Механизм сырых SQL-запросов

Возможность выполнять прямые SQL-запросы, когда ORM недостаточно.

```python
# Выполнение сырого SQL
result = session.execute("SELECT * FROM users WHERE id = :id", {"id": 1})

# Использование текстового SQL с ORM
from sqlalchemy import text
users = session.query(User).from_statement(
    text("SELECT * FROM users WHERE name LIKE :name")
).params(name="%A%").all()
```

#### 9) Механизм событий (Events)

Подписка на события жизненного цикла объектов.

```python
from sqlalchemy import event

@event.listens_for(User, 'before_insert')
def receive_before_insert(mapper, connection, target):
    target.created_at = datetime.now()
```

#### 10) Механизм гибридных атрибутов

Создание атрибутов, которые работают и как Python-свойства, и как SQL-выражения.

```python
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    __tablename__ = 'users'
    first_name = Column(String)
    last_name = Column(String)
    
    @hybrid_property
    def full_name(self):
        return f"{self.first_name} {self.last_name}"
    
    @full_name.expression
    def full_name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)

# Можно использовать в запросах
users = session.query(User).filter(User.full_name == "John Doe").all()
```

---

### 3. Работа со структурированными данными

SQLAlchemy превращает строки таблиц БД в строго типизированные Python-объекты с отслеживанием состояния.

**Преобразование результата запроса в объекты:**

```python
# ORM автоматически создает объекты из строк БД
users = session.query(User).all()
for user in users:
    print(f"User {user.name} has {len(user.posts)} posts")
```

**Работа со связанными объектами:**

```python
# Доступ к связанным объектам через relationship
user = session.query(User).first()
for post in user.posts:  # Автоматическая загрузка при необходимости
    print(post.title)
```

**Пакетные операции:**

```python
# Пакетная вставка
session.add_all([User(name=f"User{i}") for i in range(1000)])
session.commit()

# Пакетное обновление
session.query(User).filter(User.age < 18).update({"status": "minor"})
```

---

### 4. Итеративные элементы

**Автоматическое отслеживание изменений (Dirty Tracking):** Сессия отслеживает все изменения в объектах и генерирует UPDATE только для измененных полей.

```python
user = session.query(User).first()
user.name = "New Name"  # Изменение отслежено
session.commit()  # Генерируется UPDATE только для name
```

**Ленивая загрузка (Lazy Loading):** Связанные данные загружаются только при первом обращении к ним.

```python
user = session.query(User).first()  # Загружен только user
print(user.posts)  # В этот момент выполняется дополнительный запрос
```

**Автоматическое слияние (Merge):** Объединение объектов из разных сессий.

```python
detached_user = session.query(User).first()
session.close()
# Позже в другой сессии
new_session.merge(detached_user)  # Объединяет изменения
```

**Автоматическое обновление (Refresh):** Обновление объекта из БД.

```python
session.refresh(user)  # Перезагружает все поля из БД
```

---

### 5. Обработка и отладка ошибок

SQLAlchemy использует иерархию исключений для информирования о проблемах.

**Основные типы исключений:**

- `sqlalchemy.exc.SQLAlchemyError`: Базовый класс для всех исключений
- `sqlalchemy.exc.IntegrityError`: Нарушение целостности БД (NOT NULL, UNIQUE, FOREIGN KEY)
- `sqlalchemy.exc.NoResultFound`: Запрос не вернул результатов при использовании `.one()`
- `sqlalchemy.exc.MultipleResultsFound`: Запрос вернул несколько результатов при использовании `.one()`
- `sqlalchemy.exc.OperationalError`: Проблемы с подключением или синтаксисом SQL

**Структура ошибки:**

```python
try:
    user = User(name=None, email="test")  # name - NOT NULL
    session.add(user)
    session.commit()
except IntegrityError as e:
    print(f"Ошибка: {e.orig}")  # SQLite: NOT NULL constraint failed
    session.rollback()
```

**Отладка запросов:** Включение логирования всех SQL-запросов.

```python
import logging
logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

engine = create_engine('sqlite:///demo.db', echo=True)  # Простой способ
```

---

## Модуль 3. Практическое применение

### 11. Формулировка прикладной задачи

**Контекст:** При разработке приложений, работающих с базами данных, эффективное управление данными критически важно. Ручное написание SQL-запросов приводит к дублированию кода, проблемам с безопасностью и сложностям в поддержке.

**Прикладная задача:** Разработать "Систему управления библиотекой", которая демонстрирует возможности SQLAlchemy для:

1. Определения моделей данных (авторы, книги, читатели) со связями
2. Выполнения CRUD-операций через ORM
3. Оптимизации запросов через eager loading
4. Выполнения сложных запросов с агрегацией и группировкой
5. Управления транзакциями и обработки ошибок

```python
from sqlalchemy import create_engine, Column, Integer, String, Float, ForeignKey, func
from sqlalchemy.orm import declarative_base, Session, relationship, selectinload
from datetime import datetime

# 1. Определяем модели
Base = declarative_base()

class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True)
    books = relationship("Book", back_populates="author", cascade="all, delete-orphan")

class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String(100), nullable=False)
    price = Column(Float, default=0.0)
    author_id = Column(Integer, ForeignKey('authors.id', ondelete='CASCADE'))
    author = relationship("Author", back_populates="books")

# 2. Подготовка БД
engine = create_engine('sqlite:///library.db')
Base.metadata.create_all(engine)

# 3. Работа с данными
session = Session(engine)

# Создание автора с книгами
author = Author(name="Лев Толстой", email="tolstoy@example.com")
author.books = [
    Book(title="Война и мир", price=599.0),
    Book(title="Анна Каренина", price=450.0)
]
session.add(author)
session.commit()

# Запрос с оптимизацией
authors = session.query(Author).options(selectinload(Author.books)).all()
for author in authors:
    print(f"{author.name}: {len(author.books)} книг")

session.close()
```

---

### 12. Архитектура решения

Решение строится как модульное приложение с четким разделением ответственности.

**Структура проекта:**

```
library_system/
├── models/
│   ├── __init__.py
│   ├── author.py      # Модель Author
│   ├── book.py        # Модель Book
│   └── reader.py      # Модель Reader
├── repositories/
│   ├── __init__.py
│   ├── author_repo.py # Операции с авторами
│   └── book_repo.py   # Операции с книгами
├── services/
│   ├── __init__.py
│   └── library_service.py # Бизнес-логика
├── database.py        # Настройка подключения
└── main.py            # Точка входа
```

**Минимальный рабочий пример (все вместе):**

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import declarative_base, Session

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)

# Настройка БД
engine = create_engine('sqlite:///demo.db')
Base.metadata.create_all(engine)

# Работа с данными
session = Session(engine)
user = User(name="Alice")
session.add(user)
session.commit()
session.close()
```

---

### 13. Этапы реализации

**Этап 1: Настройка подключения и создание движка**
- Импорт необходимых модулей
- Создание движка для конкретной СУБД
- Настройка логирования запросов

**Этап 2: Определение базовых моделей**
- Создание класса `Base` через `declarative_base`
- Определение таблиц через классы с `__tablename__`
- Добавление колонок с типами и ограничениями

**Этап 3: Создание и управление сессией**
- Создание фабрики сессий через `sessionmaker`
- Использование контекстных менеджеров для управления сессией
- Выполнение CRUD-операций

**Этап 4: Определение отношений между моделями**
- Создание внешних ключей через `ForeignKey`
- Настройка `relationship` с `back_populates`
- Настройка каскадных операций

**Этап 5: Оптимизация запросов**
- Выявление проблемы N+1
- Использование `selectinload` и `joinedload`
- Сравнение производительности

**Этап 6: Сложные запросы и агрегация**
- Использование `func.count`, `func.sum`, `func.avg`
- Группировка через `group_by`
- Создание подзапросов

**Этап 7: Управление транзакциями**
- Явный контроль транзакций через `begin()`
- Обработка ошибок и откат изменений
- Использование контекстных менеджеров

---

### 14. Возникшие сложности и ограничения

#### Сложности работы с SQLAlchemy

##### 1. Понимание разницы между ленивой и жадной загрузкой

Новички часто сталкиваются с проблемой N+1 запросов, не понимая, что доступ к `user.posts` вызывает дополнительный запрос.

```python
# ПРОБЛЕМА: N+1 запрос
users = session.query(User).all()
for user in users:
    print(len(user.posts))  # Каждый раз отдельный SELECT!

# РЕШЕНИЕ: Жадная загрузка
users = session.query(User).options(selectinload(User.posts)).all()
for user in users:
    print(len(user.posts))  # Без дополнительных запросов
```

##### 2. Путаница между `filter` и `filter_by`

`filter` принимает выражения сравнения, а `filter_by` принимает именованные аргументы.

```python
# filter - для сложных условий
session.query(User).filter(User.name == "Alice").all()

# filter_by - для простых равенств
session.query(User).filter_by(name="Alice").all()
```

##### 3. Жизненный цикл объектов и сессий

Объекты, полученные из сессии, становятся "detached" после закрытия сессии.

```python
# ПРОБЛЕМА: Доступ к lazy-атрибуту после закрытия сессии
user = session.query(User).first()
session.close()
print(user.posts)  # DetachedInstanceError!

# РЕШЕНИЕ: Загрузить данные до закрытия
user = session.query(User).options(selectinload(User.posts)).first()
session.close()
print(user.posts)  # Работает
```

##### 4. Правильная настройка каскадных операций

Каскады могут работать неожиданно без правильной конфигурации.

```python
# Правильная настройка каскада
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", cascade="all, delete-orphan")

# Теперь при удалении пользователя удалятся и его посты
session.delete(user)
session.commit()
```

#### Ограничения проекта

##### 1. Производительность при работе с большими объемами

ORM добавляет overhead по сравнению с сырым SQL. Для массовых операций лучше использовать `bulk_insert` или сырые запросы.

```python
# Медленно для 10000+ записей
for item in items:
    session.add(Item(**item))
session.commit()

# Быстро для массовой вставки
session.bulk_insert_mappings(Item, items)
session.commit()
```

##### 2. Сложные рекурсивные запросы

Некоторые запросы (например, деревья категорий) проще написать на сыром SQL.

```python
# Рекурсивный CTE на SQLAlchemy сложен
# Иногда проще использовать сырой SQL
session.execute("""
    WITH RECURSIVE cte AS (...)
    SELECT * FROM cte
""")
```

##### 3. Отсутствие автоматической валидации типов на уровне модели

SQLAlchemy не проверяет типы данных до отправки запроса в БД.

```python
# Ошибка обнаружится только при коммите
user = User(age="not a number")  # Не вызовет ошибку
session.add(user)
session.commit()  # Ошибка здесь
```

---

### 15. Итоговая оценка инструмента

SQLAlchemy — это **промышленный стандарт** для работы с реляционными базами данных в Python. Она предоставляет наиболее полный и гибкий способ взаимодействия приложения с БД.

**Ключевые преимущества:**

- **Абстракция от СУБД:** Единый API для SQLite, PostgreSQL, MySQL, Oracle и других
- **Безопасность:** Защита от SQL-инъекций через параметризованные запросы
- **Производительность:** Оптимизация через eager loading, кэширование запросов
- **Поддерживаемость:** Самодокументируемые модели, централизованное определение схемы
- **Экосистема:** Интеграция с Alembic (миграции), Flask-SQLAlchemy, FastAPI

**Когда использовать:**

- Любое приложение, работающее с реляционной БД
- Проекты, где важна сменяемость СУБД
- Приложения со сложной логикой связей между данными
- Проекты, требующие управления миграциями схемы

**Сравнение с альтернативами:**

| Инструмент | Сильные стороны | Слабые стороны |
|------------|----------------|----------------|
| **SQLAlchemy** | Полнота, гибкость, производительность | Крутая кривая обучения |
| **Django ORM** | Простота, интеграция с Django | Менее гибкий, привязан к Django |
| **Peewee** | Легковесность, простота | Меньше возможностей |
| **PonyORM** | Синтаксис с генераторами | Меньше сообщество |
| **Raw SQL** | Максимальная производительность | Нет безопасности, сложно поддерживать |

**Вывод:**

SQLAlchemy — это незаменимый инструмент для Python-разработчиков, работающих с базами данных. Она превращает рутинную работу с SQL в элегантную работу с Python-объектами, обеспечивая безопасность, производительность и поддерживаемость кода. Несмотря на некоторую сложность освоения, SQLAlchemy является стандартом де-факто в Python-сообществе и обязательна к изучению для профессиональной разработки.
