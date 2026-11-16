# Вимоги та сценарії тестування (Library-system)

## 1. Функціональні вимоги (EARS)

Нижче наведено функціональні вимоги у класичному синтаксисі EARS, узгоджені з сутностями (`Reader`, `Author`, `Book`, `Loan`) та акторами (`Reader`, `PaymentGateway`, `EmailService`).

* **FR-01 (Пошук за назвою):** WHEN a `Reader` submits a search query by `Book.title`, THEN the System SHALL return a list of matching `Book` records including their `id`, `title`, `publication_year`, and current `status`.
* **FR-02 (Пошук за автором):** WHEN a `Reader` submits a search query by `Author.full_name`, THEN the System SHALL return all `Book` records associated with matching `Author` entities.
* **FR-03 (Видача книги):** WHEN a `Reader` requests to borrow a `Book` whose `status` is `AVAILABLE`, THEN the System SHALL create a new `Loan` record with `reader_id`, `book_id`, `borrow_date` set to the current date, `return_date` set to `null`, `fine_amount` set to `0`, and update `Book.status` to `LOANED`.
* **FR-04 (Перевірка статусу позики при поверненні):** WHEN a `Reader` initiates a book return (`UC_Return`) for an active `Loan`, THEN the System SHALL verify the `Loan` status by comparing the current date against `Loan.borrow_date` plus 14 calendar days.
* **FR-05 (Вчасне повернення книги):** IF the current date is less than or equal to `Loan.borrow_date` + 14 calendar days AND the `Book` is not reported lost, THEN the System SHALL set `Loan.return_date` to the current date, keep `Loan.fine_amount` as `0`, update `Book.status` to `AVAILABLE`, and trigger `EmailService` to send a return confirmation to `Reader.email`.
* **FR-06 (Розрахунок штрафу за прострочення):** IF the current date is greater than `Loan.borrow_date` + 14 calendar days upon return, THEN the System SHALL calculate `Loan.fine_amount` as `10 UAH * overdue_days` (where `overdue_days = current_date - (borrow_date + 14 days)`).
* **FR-07 (Реєстрація втрати книги):** WHEN a `Reader` reports a `Book` as lost during the return process (`UC_Return`), THEN the System SHALL set `Book.status` to `LOST` and set `Loan.fine_amount` to `500 UAH`.
* **FR-08 (Оплата штрафу та закриття позики):** WHILE `Loan.fine_amount` is greater than `0`, THEN the System SHALL initiate payment processing via `PaymentGateway`, and WHEN `PaymentGateway` confirms successful payment, the System SHALL set `Loan.return_date` to the current date, update `Book.status` to `AVAILABLE` (unless `Book.status` is `LOST`), and trigger `EmailService` to send a payment receipt or loss compensation confirmation to `Reader.email`.
* **FR-09 (Експорт каталогу бібліотеки — непокрита вимога):** WHEN a `Reader` requests a full catalog export in CSV format, THEN the System SHALL generate and download a file containing `Book.id`, `Book.title`, `Author.full_name`, `Book.publication_year`, and `Book.status`.

---

## 2. Нефункціональні вимоги (NFR)

* **NFR-01 (Продуктивність — час відгуку):** Система повинна обробляти 95% усіх API-запитів (пошук, оформлення позики, перевірка статусу та закриття повернення) за час, що не перевищує **300 мс**.
* **NFR-02 (Масштабованість та навантаження):** Система повинна стабільно підтримувати роботу до **50 одночасних активних користувачів** (`Reader`) без деградації часу відгуку понад встановлений поріг у 300 мс.
* **NFR-03 (Безпека платежів):** Уся передача платіжних даних між системою та зовнішнім сервісом `PaymentGateway` повинна здійснюватися виключно за захищеним протоколом **TLS 1.3**; система бібліотеки не повинна зберігати, кешувати або логувати реквізити банківських карток користувачів.
* **NFR-04 (Доступність):** Рівень доступності (Availability) системи повинен становити не менше **99.5%** у робочі години бібліотеки (з 08:00 до 21:00 за київським часом, понеділок–субота).
* **NFR-05 (Локалізація):** Інтерфейс користувача та шаблони електронних листів, що формуються для відправки через `EmailService`, повинні підтримувати **2 мови**: українську (`uk`) та англійську (`en`).
* **NFR-06 (Сумісність та адаптивність):** Вебінтерфейс системи повинен коректно відображатися та функціонувати в актуальних версіях браузерів **Google Chrome, Mozilla Firefox та Apple Safari** на екранах шириною від **360 px** (мобільні пристрої) до **1920 px** (десктопні монітори).

---

## 3. Сценарії Gherkin для Use Case `UC_Return`

```gherkin
Feature: UC_Return — Повернення книги та врегулювання штрафів
  As a Reader (Читач)
  I want to return a borrowed book, pay any applicable overdue or loss fines, and receive email confirmation
  So that my loan record is properly closed and I have proof of return or payment.

  Background:
    Given the Reader "Олена Коваленко" with id "r-101" and email "olena.kovalenko@example.com" is authenticated
    And the Book "Тіні забутих предків" with id "b-202" has status "LOANED"
    And an active Loan with id "l-303" exists for reader_id "r-101" and book_id "b-202" with borrow_date "2026-11-01" and return_date null

  Scenario: Вчасне повернення книги без нарахування штрафу
    Given the current date is "2026-11-12" (11 days since borrow_date, within the 14-day limit)
    When the Reader initiates the return process for Loan "l-303"
    Then the System checks the Loan status and confirms there is no overdue period
    And the System sets Loan "l-303" return_date to "2026-11-12" and fine_amount to 0
    And the System updates Book "b-202" status from "LOANED" to "AVAILABLE"
    And the EmailService sends a return confirmation email to "olena.kovalenko@example.com" for Book "Тіні забутих предків"

  Scenario: Повернення книги з простроченням та оплатою штрафу через PaymentGateway
    Given the current date is "2026-11-20" (19 days since borrow_date, which is 5 overdue days)
    When the Reader initiates the return process for Loan "l-303"
    Then the System checks the Loan status and calculates fine_amount as 50 UAH (5 days * 10 UAH)
    And the System requests fine payment of 50 UAH from the PaymentGateway
    When the PaymentGateway processes the transaction and returns payment confirmation for 50 UAH
    Then the System sets Loan "l-303" return_date to "2026-11-20" and records fine_amount as 50
    And the System updates Book "b-202" status from "LOANED" to "AVAILABLE"
    And the EmailService sends a payment receipt email for 50 UAH to "olena.kovalenko@example.com"

  Scenario: Повідомлення про втрату книги, оплата компенсації та закриття позики
    Given the current date is "2026-11-16"
    When the Reader initiates the return process for Loan "l-303" and reports Book "b-202" as lost
    Then the System updates Book "b-202" status to "LOST"
    And the System sets Loan "l-303" fine_amount to 500 UAH
    And the System requests loss compensation payment of 500 UAH from the PaymentGateway
    When the PaymentGateway processes the transaction and returns payment confirmation for 500 UAH
    Then the System sets Loan "l-303" return_date to "2026-11-16" and keeps Book "b-202" status as "LOST"
    And the EmailService sends a loan closure and loss compensation confirmation email for 500 UAH to "olena.kovalenko@example.com"
```

---

## 4. Матриця трасування (Traceability Matrix)

Матриця демонструє зв'язки багато-до-багатьох ($N:M$) між функціональними вимогами (`FR`), прецедентами (`Use Cases`) та сценаріями тестування (`Gherkin Scenarios`). Нефункціональні вимоги (`NFR`) виключені з матриці згідно зі специфікацією.

| ID вимоги (FR) | Короткий опис (EARS) | Use Case(s) | Gherkin Scenario(s) (`UC_Return`) | Статус покриття та примітки |
| :--- | :--- | :--- | :--- | :--- |
| **FR-01** | Пошук книги за назвою (`Book.title`) | `UC_Search`, `UC_SearchByTitle` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-02** | Пошук книги за автором (`Author.full_name`) | `UC_Search`, `UC_SearchByAuthor` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-03** | Видача книги (`status: AVAILABLE` $\rightarrow$ `LOANED`) | `UC_Borrow` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-04** | Перевірка статусу та терміну позики (14 днів) | `UC_Return`, `UC_CheckLoanStatus` | • Вчасне повернення книги<br>• Повернення книги з простроченням | Повністю покрито |
| **FR-05** | Вчасне повернення книги та лист від `EmailService` | `UC_Return`, `UC_SendNotification` | • Вчасне повернення книги | Повністю покрито |
| **FR-06** | Розрахунок штрафу за прострочення (`10 грн/день`) | `UC_Return`, `UC_CheckLoanStatus`, `UC_PayFine` | • Повернення книги з простроченням | Повністю покрито |
| **FR-07** | Реєстрація втрати книги (`status: LOST`, штраф `500 грн`) | `UC_Return`, `UC_PayFine` | • Повідомлення про втрату книги | Повністю покрито |
| **FR-08** | Оплата штрафу через `PaymentGateway` та чек через `EmailService` | `UC_Return`, `UC_PayFine`, `UC_SendNotification` | • Повернення книги з простроченням<br>• Повідомлення про втрату книги | Повністю покрито |
| **FR-09** | Експорт повного каталогу книг у формат CSV | — | — | **Непокрита вимога:** функціонал експорту даних не входить до базових прецедентів поточного релізу (`UC_Search`, `UC_Borrow`, `UC_Return`) і запланований на наступну ітерацію розробки. |