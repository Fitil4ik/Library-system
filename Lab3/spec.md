# Взаємодія Sequence Diagrams (Library-system)

## Намір
Описати внутрішню взаємодію компонентів системи управління бібліотекою для кожного з трьох сценаріїв прецеденту повернення книги (`UC_Return`).

## Межі
- В межах (In scope): виключно внутрішня взаємодія об'єктів та зовнішніх сервісів для 3 сценаріїв `UC_Return` (включно з `UC_CheckLoanStatus`, `UC_PayFine`, `UC_SendNotification`).
- Поза межами (Out of scope): прецеденти пошуку (`UC_Search`), видачі (`UC_Borrow`) та експорту каталогу в CSV (`FR-10`) у цьому завданні не моделюються.

## Актори
- Зовнішні актори:
  - `Reader` - читач, який ініціює повернення або повідомляє про втрату книги.
  - `PaymentGateway` - зовнішній платіжний шлюз для оплати штрафів.
  - `EmailService` - зовнішній поштовий сервіс для відправки листів-підтверджень.
- Компоненти системи:
  - `ReturnController` - приймає запити від читача та підтвердження від платіжного шлюзу.
  - `LoanService` - реалізує бізнес-логіку перевірки термінів, розрахунку штрафів та закриття позики.
  - `LoanRepository` - працює з сутністю `Loan (id, reader_id, book_id, borrow_date, return_date, fine_amount)`.
  - `BookRepository` - працює з сутністю `Book (id, title, publication_year, status [AVAILABLE, LOANED, LOST])`.
  - `ReaderRepository` - працює з сутністю `Reader (id, email, full_name)` для отримання адреси `Reader.email`.

## Сценарії
1. Вчасне повернення книги:
   - `Reader` ініціює повернення активної позики (`loanId`).
   - Система отримує дані позики, перевіряє термін (`current_date <= Loan.return_date`), залишає `fine_amount = 0.00`.
   - Система фіксує фактичну `Loan.return_date`, оновлює `Book.status` з `LOANED` на `AVAILABLE` та надсилає лист-підтвердження на `Reader.email` через `EmailService`.
2. Повернення книги з простроченням та оплатою штрафу:
   - `Reader` ініціює повернення позики, де `current_date > Loan.return_date`.
   - Система рахує штраф (`10.00 грн * overdue_days`) і записує його в `Loan.fine_amount`.
   - Система ініціює оплату через `PaymentGateway`; після підтвердження транзакції фіксує фактичну `Loan.return_date`, оновлює `Book.status` на `AVAILABLE` та відправляє квитанцію на `Reader.email` через `EmailService`.
3. Повідомлення про втрату книги з компенсацією:
   - `Reader` ініціює закриття позики і повідомляє про втрату книги.
   - Система одразу змінює `Book.status` на `LOST` і встановлює фіксований штраф `500.00 грн` у `Loan.fine_amount` (перекриваючи можливу пеню за прострочення).
   - Система проводить оплату `500.00 грн` через `PaymentGateway`, після підтвердження фіксує фактичну `Loan.return_date` (залишаючи `Book.status = LOST`) та надсилає підтвердження на `Reader.email` через `EmailService`.

## Критерії прийняття
1. Окрема Mermaid `sequenceDiagram` на кожен із 3 сценаріїв у файлі `Lab3/diagrams.md`.
2. Усі виклики оформлені з реалістичними назвами методів (наприклад, `initiateReturn(loanId)`, `getActiveLoan(loanId)`, `calculateFine(loan, currentDate)`, `processPayment(loanId, amount)`, `updateBookStatus(bookId, status)`, `getReaderEmail(readerId)`, `sendConfirmation(email, payload)`) та явними return-повідомленнями (`-->>`).
3. Кожна діаграма містить щонайменше один комбінований фрагмент (`alt`, `opt` або `loop`) для умовних розгалужень, перевірки результатів або повторних спроб.