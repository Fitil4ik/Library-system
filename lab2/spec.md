Специфікація ER-моделі (Library-system)

Критерії прийняття (Задача для AI):

    Згенеруй Use Case діаграму в синтаксисі PlantUML або Mermaid.
    Діаграма має містити мінімум 2 типи акторів (людина та зовнішня система/сервіс).
    Використай зв'язки include, extend та узагальнення (generalization) на Use Case діаграмі.
    Напиши вимоги у синтаксисі EARS (наприклад, WHEN <тригер>, система SHALL <реакція> / WHILE <стан>...) для основних процесів.
    Напиши User stories для основних процесів.
    Напиши сценарії Gherkin (Given/When/Then) для цих же процесів.
    Створи матрицю трасування (вимога - сценарій / UC) у форматі Markdown таблиці. Зв'язки мають бути багато-до-багатьох (не 1:1), залиш мінімум 1 непокриту вимогу.

Актори:

    Reader (Читач) - людина.
    PaymentGateway (Платіжний шлюз) - зовнішня система.
    EmailService (Сервіс розсилок) - зовнішня система.


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
    status (enum) - AVAILABLE, LOANED, LOST

Loan (Видача книги) - це асоціативна сутність, зв'язок оренди несе власні атрибути.

    id (uuid) - primary key
    reader_id (uuid) - foreign key
    book_id (uuid) - foreign key
    borrow_date (date)
    return_date (date)
    fine_amount (decimal)