# Вимоги та сценарії тестування (Library-system)

## 1. Функціональні вимоги (EARS)

Нижче наведено атомарні функціональні вимоги у класичному синтаксисі EARS (один тригер `WHEN` / `IF` / `WHILE` на одне речення `THEN the System SHALL`), узгоджені з сутностями (`Reader`, `Author`, `Book`, `Loan`) та бізнес-правилами зі специфікації.

* **FR-01 (Пошук за назвою):** WHEN a `Reader` submits a search query by `Book.title`, THEN the System SHALL return a list of matching `Book` records including their `id`, `title`, `publication_year`, and current `status`.
* **FR-02 (Пошук за автором):** WHEN a `Reader` submits a search query by `Author.full_name`, THEN the System SHALL return all `Book` records associated with matching `Author` entities.
* **FR-03 (Видача книги):** WHEN a `Reader` requests to borrow a `Book` whose `status` is `AVAILABLE`, THEN the System SHALL create a new `Loan` record with `reader_id`, `book_id`, `borrow_date` set to the current date, `return_date` set to `borrow_date` plus 14 calendar days, `fine_amount` set to `0.00`, and update `Book.status` to `LOANED`.
* **FR-04 (Перевірка статусу позики при поверненні):** WHEN a `Reader` initiates a book return (`UC_Return`) for an active `Loan`, THEN the System SHALL verify the `Loan` status by comparing the current date against the scheduled `Loan.return_date`.
* **FR-05 (Вчасне повернення книги):** IF the current date is less than or equal to `Loan.return_date` AND the `Book` is not reported lost, THEN the System SHALL set `Loan.return_date` to the actual return date, keep `Loan.fine_amount` as `0.00`, update `Book.status` to `AVAILABLE`, and trigger `EmailService` to send a return confirmation to `Reader.email`.
* **FR-06 (Розрахунок штрафу за прострочення):** IF the current date is greater than `Loan.return_date` AND the `Book` is not reported lost, THEN the System SHALL calculate `Loan.fine_amount` as `10.00 UAH * overdue_days` (where `overdue_days = current_date - Loan.return_date`).
* **FR-07 (Реєстрація втрати книги та пріоритет штрафів):** WHEN a `Reader` reports a `Book` as lost during `UC_Return`, THEN the System SHALL update `Book.status` to `LOST` and set `Loan.fine_amount` to a fixed `500.00 UAH`, overriding any overdue daily fine calculation regardless of whether `current_date` exceeds `Loan.return_date`.
* **FR-08 (Ініціація оплати штрафу):** WHILE `Loan.fine_amount` is greater than `0.00`, THEN the System SHALL initiate a payment transaction for `Loan.fine_amount` via `PaymentGateway`.
* **FR-09 (Підтвердження оплати та закриття позики):** WHEN `PaymentGateway` confirms successful payment for a `Loan`, THEN the System SHALL set `Loan.return_date` to the current date, update `Book.status` to `AVAILABLE` (unless `Book.status` is `LOST`), and trigger `EmailService` to send a payment receipt or loss compensation confirmation to `Reader.email`.
* **FR-10 (Експорт каталогу бібліотеки — непокрита вимога):** WHEN a `Reader` requests a full catalog export in CSV format, THEN the System SHALL generate and download a file containing `Book.id`, `Book.title`, `Author.full_name`, `Book.publication_year`, and `Book.status`.

---

## 2. Нефункціональні вимоги (NFR)

* **NFR-01 (Продуктивність — час відгуку):** Система повинна обробляти 95% усіх запитів при обробці повернення книги та розрахунку штрафу за час, що не перевищує **300 мс**.
* **NFR-02 (Пропускна здатність):** Система повинна стабільно витримувати навантаження до **50 одночасних запитів читачів** (`Reader`) без деградації часу відгуку понад 300 мс.
* **NFR-03 (Безпека платежів):** Передача даних до зовнішнього платіжного шлюзу `PaymentGateway` повинна здійснюватися виключно за протоколом **TLS 1.3** без збереження, кешування або логування карткових даних користувачів у системі бібліотеки.
* **NFR-04 (Доступність):** Рівень доступності (Availability) системи повинен становити не менше **99.5%** у робочі години бібліотеки.
* **NFR-05 (Локалізація та пристрої):** Інтерфейс системи та сповіщення `EmailService` повинні підтримувати **українську та англійську мови**, а вебінтерфейс повинен коректно працювати у браузерах **Chrome, Firefox, Safari** на екранах шириною від **360px до 1920px**.

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
    And an active Loan with id "l-303" exists for reader_id "r-101" and book_id "b-202" with borrow_date "2026-11-01", scheduled return_date "2026-11-15", and fine_amount 0.00

  Scenario: Вчасне повернення книги без нарахування штрафу
    Given the current date is "2026-11-12" (within the 14-day period before return_date "2026-11-15")
    When the Reader initiates the return process for Loan "l-303"
    Then the System checks the Loan status and confirms there is no overdue period
    And the System sets Loan "l-303" return_date to "2026-11-12" and keeps fine_amount as 0.00
    And the System updates Book "b-202" status from "LOANED" to "AVAILABLE"
    And the EmailService sends a return confirmation email to "olena.kovalenko@example.com" for Book "Тіні забутих предків"

  Scenario: Повернення книги з простроченням та оплатою штрафу через PaymentGateway
    Given the current date is "2026-11-20" (5 overdue days after scheduled return_date "2026-11-15")
    When the Reader initiates the return process for Loan "l-303"
    Then the System checks the Loan status and calculates fine_amount as 50.00 UAH (5 days * 10.00 UAH)
    And the System initiates a payment transaction of 50.00 UAH via the PaymentGateway
    When the PaymentGateway processes the transaction and returns payment confirmation for 50.00 UAH
    Then the System sets Loan "l-303" return_date to "2026-11-20" and records fine_amount as 50.00
    And the System updates Book "b-202" status from "LOANED" to "AVAILABLE"
    And the EmailService sends a payment receipt email for 50.00 UAH to "olena.kovalenko@example.com"

  Scenario: Повідомлення про втрату книги з простроченням та пріоритетом фіксованої компенсації
    Given the current date is "2026-11-20" (5 days past scheduled return_date "2026-11-15")
    When the Reader initiates the return process for Loan "l-303" and reports Book "b-202" as lost
    Then the System updates Book "b-202" status to "LOST"
    And the System sets Loan "l-303" fine_amount to 500.00 UAH and does not add overdue daily penalties (0.00 UAH overdue fee)
    And the System initiates a loss compensation payment of 500.00 UAH via the PaymentGateway
    When the PaymentGateway processes the transaction and returns payment confirmation for 500.00 UAH
    Then the System sets Loan "l-303" return_date to "2026-11-20" and keeps Book "b-202" status as "LOST"
    And the EmailService sends a loan closure and loss compensation confirmation email for 500.00 UAH to "olena.kovalenko@example.com"
```

---

## 4. Матриця трасування (Traceability Matrix)

Матриця демонструє зв'язки багато-до-багатьох ($N:M$) між функціональними вимогами (`FR`), прецедентами (`Use Cases`) та сценаріями тестування (`Gherkin Scenarios`). Нефункціональні вимоги (`NFR`) виключені з матриці згідно зі специфікацією.

| ID вимоги (FR) | Короткий опис (EARS) | Use Case(s) | Gherkin Scenario(s) (`UC_Return`) | Статус покриття та примітки |
| :--- | :--- | :--- | :--- | :--- |
| **FR-01** | Пошук книги за назвою (`Book.title`) | `UC_Search`, `UC_SearchByTitle` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-02** | Пошук книги за автором (`Author.full_name`) | `UC_Search`, `UC_SearchByAuthor` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-03** | Видача книги на 14 днів (`AVAILABLE` $\rightarrow$ `LOANED`) | `UC_Borrow` | — | Покрито на рівні Use Case (поза межами `UC_Return`) |
| **FR-04** | Перевірка статусу та планової дати повернення позики | `UC_Return`, `UC_CheckLoanStatus` | • Вчасне повернення книги<br>• Повернення книги з простроченням | Повністю покрито |
| **FR-05** | Вчасне повернення книги та лист від `EmailService` | `UC_Return`, `UC_SendNotification` | • Вчасне повернення книги | Повністю покрито |
| **FR-06** | Розрахунок штрафу за прострочення (`10.00 грн/день`) | `UC_Return`, `UC_CheckLoanStatus`, `UC_PayFine` | • Повернення книги з простроченням | Повністю покрито |
| **FR-07** | Реєстрація втрати (`status: LOST`, `500.00 грн`, пріоритет над пенею) | `UC_Return`, `UC_PayFine` | • Повідомлення про втрату книги з простроченням | Повністю покрито |
| **FR-08** | Ініціація оплати штрафу через `PaymentGateway` (`fine_amount > 0`) | `UC_Return`, `UC_PayFine` | • Повернення книги з простроченням<br>• Повідомлення про втрату книги з простроченням | Повністю покрито |
| **FR-09** | Обробка підтвердження оплати, закриття позики та лист `EmailService` | `UC_Return`, `UC_PayFine`, `UC_SendNotification` | • Повернення книги з простроченням<br>• Повідомлення про втрату книги з простроченням | Повністю покрито |
| **FR-10** | Експорт повного каталогу книг у формат CSV | — | — | **Непокрита вимога:** функціонал експорту даних не входить до базових прецедентів поточного релізу (`UC_Search`, `UC_Borrow`, `UC_Return`) і запланований на наступну ітерацію розробки. |