from sqlalchemy import create_engine, Column, Integer, String, Float, ForeignKey, func
from sqlalchemy.orm import declarative_base, Session, relationship, selectinload
from datetime import datetime
import time

# ============= 1. ОПРЕДЕЛЕНИЕ МОДЕЛЕЙ =============
Base = declarative_base()


class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True)

    # Связь с книгами (один ко многим)
    books = relationship("Book", back_populates="author", cascade="all, delete-orphan")

    def __repr__(self):
        return f"<Author {self.name}>"


class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String(100), nullable=False)
    price = Column(Float, default=0.0)
    author_id = Column(Integer, ForeignKey('authors.id', ondelete='CASCADE'))

    author = relationship("Author", back_populates="books")

    def __repr__(self):
        return f"<Book '{self.title}', ${self.price}>"


# ============= 2. ПОДГОТОВКА БАЗЫ ДАННЫХ =============
engine = create_engine('sqlite:///library.db', echo=False)
Base.metadata.drop_all(engine)  # Очистка для демо
Base.metadata.create_all(engine)  # Создание таблиц


# ============= 3. ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ =============
def print_separator(title):
    print(f"\n{'=' * 50}")
    print(f"📚 {title}")
    print('=' * 50)


def timer(func):
    """Декоратор для замера времени"""

    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"⏱ Время выполнения: {time.time() - start:.4f} сек")
        return result

    return wrapper


# ============= 4. ДЕМОНСТРАЦИЯ ВОЗМОЖНОСТЕЙ =============
@timer
def demo_create_data():
    """Создание данных с использованием связей"""
    print_separator("СОЗДАНИЕ ДАННЫХ")

    session = Session(engine)

    # Создаем авторов
    author1 = Author(name="Лев Толстой", email="tolstoy@example.com")
    author2 = Author(name="Федор Достоевский", email="dostoevsky@example.com")

    # Создаем книги и связываем с авторами
    author1.books = [
        Book(title="Война и мир", price=599.0),
        Book(title="Анна Каренина", price=450.0),
        Book(title="Воскресение", price=399.0)
    ]

    author2.books = [
        Book(title="Преступление и наказание", price=550.0),
        Book(title="Идиот", price=480.0),
        Book(title="Братья Карамазовы", price=650.0)
    ]

    # Сохраняем в БД (каскадно сохранятся и книги)
    session.add_all([author1, author2])
    session.commit()

    print(f"✅ Добавлено авторов: {session.query(Author).count()}")
    print(f"✅ Добавлено книг: {session.query(Book).count()}")

    session.close()


@timer
def demo_query_basic():
    """Базовые запросы"""
    print_separator("БАЗОВЫЕ ЗАПРОСЫ")

    session = Session(engine)

    # SELECT * FROM authors
    all_authors = session.query(Author).all()
    print(f"\nВсе авторы ({len(all_authors)}):")
    for author in all_authors:
        print(f"  • {author.name} ({author.email})")

    # SELECT с фильтрацией
    tolstoy = session.query(Author).filter(Author.name == "Лев Толстой").first()
    print(f"\n🔍 Найден автор: {tolstoy.name}")

    # SELECT с сортировкой
    expensive_books = session.query(Book).order_by(Book.price.desc()).limit(3).all()
    print(f"\n💰 Самые дорогие книги:")
    for book in expensive_books:
        print(f"  • {book.title} - {book.price} руб.")

    session.close()


@timer
def demo_relationships():
    """Работа со связями и N+1 проблема"""
    print_separator("РАБОТА СО СВЯЗЯМИ")

    session = Session(engine)

    # ===== LAZY LOADING (создает проблему N+1) =====
    print("\n🐌 LAZY LOADING (N+1 запросов):")
    authors = session.query(Author).all()
    for author in authors:
        # Каждый author.books вызывает отдельный SELECT!
        books_count = len(author.books)
        print(f"  {author.name}: {books_count} книг")

    # ===== EAGER LOADING (решает проблему) =====
    print("\n⚡ EAGER LOADING (1 запрос):")
    authors = session.query(Author).options(selectinload(Author.books)).all()
    for author in authors:
        books_count = len(author.books)  # Данные уже загружены
        print(f"  {author.name}: {books_count} книг")

    session.close()


@timer
def demo_aggregation():
    """Агрегатные функции и группировка"""
    print_separator("АГРЕГАЦИЯ И ГРУППИРОВКА")

    session = Session(engine)

    # Общая статистика по книгам
    stats = session.query(
        func.count(Book.id).label('total_books'),
        func.avg(Book.price).label('avg_price'),
        func.min(Book.price).label('min_price'),
        func.max(Book.price).label('max_price'),
        func.sum(Book.price).label('total_value')
    ).first()

    print(f"\n📊 Статистика по библиотеке:")
    print(f"  Всего книг: {stats.total_books}")
    print(f"  Средняя цена: {stats.avg_price:.2f} руб.")
    print(f"  Минимальная цена: {stats.min_price} руб.")
    print(f"  Максимальная цена: {stats.max_price} руб.")
    print(f"  Общая стоимость: {stats.total_value} руб.")

    # Группировка по авторам
    print(f"\n📖 Книги по авторам:")
    author_stats = session.query(
        Author.name,
        func.count(Book.id).label('book_count'),
        func.avg(Book.price).label('avg_price')
    ).join(Book).group_by(Author.id).all()

    for author_name, book_count, avg_price in author_stats:
        print(f"  {author_name}: {book_count} книг, средняя цена {avg_price:.0f} руб.")

    session.close()


@timer
def demo_update_delete():
    """Обновление и удаление данных"""
    print_separator("ОБНОВЛЕНИЕ И УДАЛЕНИЕ")

    session = Session(engine)

    # UPDATE
    print("\n✏️ Обновление цены книг Толстого (+10%):")
    tolstoy = session.query(Author).filter(Author.name == "Лев Толстой").first()
    for book in tolstoy.books:
        old_price = book.price
        book.price = round(book.price * 1.1, 2)
        print(f"  '{book.title}': {old_price} → {book.price} руб.")
    session.commit()

    # DELETE (каскадное удаление)
    print(f"\n🗑 Удаление автора 'Федор Достоевский' и всех его книг:")
    dostoevsky = session.query(Author).filter(Author.name == "Федор Достоевский").first()
    session.delete(dostoevsky)
    session.commit()

    print(f"  Осталось авторов: {session.query(Author).count()}")
    print(f"  Осталось книг: {session.query(Book).count()}")

    session.close()


@timer
def demo_complex_query():
    """Сложный запрос с подзапросом"""
    print_separator("СЛОЖНЫЙ ЗАПРОС")

    session = Session(engine)

    # Найти книги дороже средней цены своего автора
    # (показываем книги, которые дороже среднего по автору)

    from sqlalchemy import and_

    # Подзапрос: средняя цена книг для каждого автора
    subquery = session.query(
        Book.author_id,
        func.avg(Book.price).label('avg_author_price')
    ).group_by(Book.author_id).subquery()

    # Основной запрос
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

    print("\n📈 Книги дороже средней цены своего автора:")
    if expensive_books:
        for author, title, price, avg_price in expensive_books:
            print(f"  • '{title}' ({author}) - {price} руб. > среднее {avg_price:.0f} руб.")
    else:
        print("  Нет таких книг")

    session.close()


# ============= 5. ГЛАВНАЯ ФУНКЦИЯ =============
def main():
    print("\n" + "🎯" * 25)
    print("SQLALCHEMY ДЕМОНСТРАЦИОННЫЙ СТЕНД")
    print("🎯" * 25)

    demo_create_data()  # Создание данных
    demo_query_basic()  # Базовые запросы
    demo_relationships()  # Связи и N+1 проблема
    demo_aggregation()  # Агрегация и группировка
    demo_update_delete()  # Обновление и удаление
    demo_complex_query()  # Сложные запросы

    print("\n" + "✨" * 25)
    print("ДЕМОНСТРАЦИЯ ЗАВЕРШЕНА")
    print("✨" * 25)


if __name__ == "__main__":
    main()
