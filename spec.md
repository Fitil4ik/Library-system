Специфікація ER-моделі (Library-system)

Критерії прийняття (Задача для AI):

    Згенеруй Mermaid erDiagram на основі наведених нижче сутностей.
    Всі ідентифікатори (id) повинні мати однаковий тип даних - uuid.
    Зв'язок між книгами та авторами має бути чистим зв'язком багато-до-багатьох без створення фізичної сполучної таблиці.
    Модель повинна бути нормалізована.
    Додай поле статусу в сутність Book.
    Поле статусу в сутності Book має бути чітко обмежене конкретними значеннями (enum: AVAILABLE, LOANED, LOST), а не використовуватися як базовий тип string.

Сутності та атрибути:
Reader (Читач)

    id (uuid) - primary key
    email (string)
    full_name (string)

Author (Автор)

    id (uuid) - primary key
    full_name (string)

Book (Книга)

    id (uuid) - primary key
    title (string)
    publication_year (number)

Loan (Видача книги) - це асоціативна сутність, зв'язок оренди несе власні атрибути.

    id (uuid) - primary key
    reader_id (uuid) - foreign key
    book_id (uuid) - foreign key
    borrow_date (date)
    return_date (date)