# Sequence-діаграми взаємодії для прецеденту `UC_Return`

Документ описує внутрішню взаємодію компонентів системи управління бібліотекою та зовнішніх сервісів для трьох сценаріїв повернення книги (`UC_Return`), включно з перевіркою статусу позики (`UC_CheckLoanStatus`), оплатою штрафів/компенсацій (`UC_PayFine`) та надсиланням сповіщень (`UC_SendNotification`).

## Учасники взаємодії

1. `Reader` — зовнішній актор (читач), який ініціює повернення книги або повідомляє про її втрату.
2. `ReturnController` — контролер вхідних запитів від клієнтського інтерфейсу.
3. `LoanService` — сервіс бізнес-логіки перевірки термінів, нарахування штрафів, координації оплат та закриття позик.
4. `LoanRepository` — репозиторій сутності `Loan (id, reader_id, book_id, borrow_date, return_date, fine_amount)`.
5. `BookRepository` — репозиторій сутності `Book (id, title, publication_year, status [AVAILABLE, LOANED, LOST])`.
6. `ReaderRepository` — репозиторій сутності `Reader (id, email, full_name)`.
7. `PaymentGateway` — зовнішній платіжний шлюз для проведення транзакцій.
8. `EmailService` — зовнішній поштовий сервіс для відправки підтверджень та квитанцій.

## 1. Сценарій 1: Вчасне повернення книги

Читач ініціює повернення книги, а контролер передає запит у спільний метод `LoanService.processReturn(loanId)`. Після отримання даних позики сервіс перевіряє термін (`current_date <= Loan.return_date`), залишає штраф нульовим (`0.00 грн`), переводить примірник у статус `AVAILABLE` та надсилає лист-підтвердження на електронну пошту читача (за її наявності).

```mermaid
sequenceDiagram
    autonumber
    actor Reader
    participant ReturnController
    participant LoanService
    participant LoanRepository
    participant BookRepository
    participant ReaderRepository
    participant PaymentGateway
    participant EmailService

    Reader->>ReturnController: initiateReturn(loanId)
    activate ReturnController

    ReturnController->>LoanService: processReturn(loanId)
    activate LoanService

    LoanService->>LoanRepository: getActiveLoan(loanId)
    activate LoanRepository
    LoanRepository-->>LoanService: loan
    deactivate LoanRepository

    alt Термін повернення дотримано: current_date <= Loan.return_date
        LoanService->>LoanService: setFineAmount(loan, 0.00)
        activate LoanService
        LoanService-->>LoanService: fineAmount = 0.00
        deactivate LoanService

        LoanService->>LoanRepository: closeLoan(loanId, currentDate, 0.00)
        activate LoanRepository
        LoanRepository-->>LoanService: closedLoan
        deactivate LoanRepository

        LoanService->>BookRepository: updateBookStatus(loan.book_id, AVAILABLE)
        activate BookRepository
        BookRepository-->>LoanService: updatedBook
        deactivate BookRepository

        LoanService->>ReaderRepository: getReaderEmail(loan.reader_id)
        activate ReaderRepository
        ReaderRepository-->>LoanService: readerEmail
        deactivate ReaderRepository

        opt Адресу електронної пошти знайдено: readerEmail != null
            LoanService->>EmailService: sendConfirmation(readerEmail, returnConfirmationPayload)
            activate EmailService
            EmailService-->>LoanService: deliveryStatus (SENT)
            deactivate EmailService
        end

        LoanService-->>ReturnController: ReturnResult(SUCCESS, fineAmount=0.00)
    else Позику не знайдено або термін прострочено: current_date > Loan.return_date
        LoanService-->>ReturnController: ReturnResult(INVALID_OR_OVERDUE, loanId)
    end

    deactivate LoanService
    ReturnController-->>Reader: ReturnResponse(status, fineAmount)
    deactivate ReturnController
```

## 2. Сценарій 2: Повернення книги з простроченням та оплатою штрафу

Контролер викликає спільний метод `LoanService.processReturn(loanId)`. Виявивши прострочення (`current_date > Loan.return_date`), сервіс обчислює пеню (`10.00 грн * overdue_days`), зберігає її в `Loan.fine_amount` та здійснює прямий запит на оплату до `PaymentGateway` із циклом повторних спроб при таймауті (до 3 спроб). Після підтвердження платежу позика закривається, книга повертається в статус `AVAILABLE`, а на пошту читача (за наявності `Reader.email`) надсилається квитанція.

```mermaid
sequenceDiagram
    autonumber
    actor Reader
    participant ReturnController
    participant LoanService
    participant LoanRepository
    participant BookRepository
    participant ReaderRepository
    participant PaymentGateway
    participant EmailService

    Reader->>ReturnController: initiateReturn(loanId)
    activate ReturnController

    ReturnController->>LoanService: processReturn(loanId)
    activate LoanService

    LoanService->>LoanRepository: getActiveLoan(loanId)
    activate LoanRepository
    LoanRepository-->>LoanService: loan
    deactivate LoanRepository

    LoanService->>LoanService: calculateFine(loan, currentDate)
    activate LoanService
    LoanService-->>LoanService: fineAmount (10.00 * overdue_days)
    deactivate LoanService

    LoanService->>LoanRepository: saveCalculatedFine(loanId, fineAmount)
    activate LoanRepository
    LoanRepository-->>LoanService: loanWithFine
    deactivate LoanRepository

    loop Спроба оплати через шлюз: до 3 спроб при таймауті
        LoanService->>PaymentGateway: processPayment(loanId, fineAmount)
        activate PaymentGateway
        PaymentGateway-->>LoanService: paymentResponse (CONFIRMED | DECLINED | TIMEOUT)
        deactivate PaymentGateway
    end

    alt Оплату підтверджено: paymentResponse.status == CONFIRMED
        LoanService->>LoanRepository: closeLoan(loanId, currentDate, fineAmount)
        activate LoanRepository
        LoanRepository-->>LoanService: closedLoan
        deactivate LoanRepository

        LoanService->>BookRepository: updateBookStatus(loan.book_id, AVAILABLE)
        activate BookRepository
        BookRepository-->>LoanService: updatedBook
        deactivate BookRepository

        LoanService->>ReaderRepository: getReaderEmail(loan.reader_id)
        activate ReaderRepository
        ReaderRepository-->>LoanService: readerEmail
        deactivate ReaderRepository

        opt Адресу електронної пошти знайдено: readerEmail != null
            LoanService->>EmailService: sendConfirmation(readerEmail, fineReceiptPayload)
            activate EmailService
            EmailService-->>LoanService: deliveryStatus (SENT)
            deactivate EmailService
        end

        LoanService-->>ReturnController: ReturnResult(SUCCESS, fineAmount, paymentResponse.transactionId)
    else Оплату відхилено або вичерпано ліміт спроб: DECLINED | TIMEOUT
        LoanService-->>ReturnController: ReturnResult(PAYMENT_FAILED, fineAmount)
    end

    deactivate LoanService
    ReturnController-->>Reader: ReturnResponse(status, fineAmount, receiptInfo)
    deactivate ReturnController
```

## 3. Сценарій 3: Повідомлення про втрату книги з компенсацією

Читач повідомляє про втрату книги (`reportLostBook(loanId)`). Система негайно переводить примірник у статус `LOST`, розраховує фіксовану компенсацію `500.00 грн` (перекриваючи можливу пеню за прострочення) та зберігає її в `Loan.fine_amount`. Після проведення оплати через `PaymentGateway` позика закривається (книга залишається у статусі `LOST`), а підтвердження надсилається на `Reader.email` за наявності адреси.

```mermaid
sequenceDiagram
    autonumber
    actor Reader
    participant ReturnController
    participant LoanService
    participant LoanRepository
    participant BookRepository
    participant ReaderRepository
    participant PaymentGateway
    participant EmailService

    Reader->>ReturnController: reportLostBook(loanId)
    activate ReturnController

    ReturnController->>LoanService: processLostBook(loanId)
    activate LoanService

    LoanService->>LoanRepository: getActiveLoan(loanId)
    activate LoanRepository
    LoanRepository-->>LoanService: loan
    deactivate LoanRepository

    LoanService->>BookRepository: updateBookStatus(loan.book_id, LOST)
    activate BookRepository
    BookRepository-->>LoanService: lostBook
    deactivate BookRepository

    LoanService->>LoanService: calculateLostBookFine(loan)
    activate LoanService
    LoanService-->>LoanService: fineAmount = 500.00
    deactivate LoanService

    LoanService->>LoanRepository: saveCalculatedFine(loanId, fineAmount)
    activate LoanRepository
    LoanRepository-->>LoanService: loanWithCompensation
    deactivate LoanRepository

    LoanService->>PaymentGateway: processPayment(loanId, fineAmount)
    activate PaymentGateway
    PaymentGateway-->>LoanService: paymentResponse (CONFIRMED | DECLINED)
    deactivate PaymentGateway

    alt Оплату компенсації підтверджено: paymentResponse.status == CONFIRMED
        LoanService->>LoanRepository: closeLoan(loanId, currentDate, fineAmount)
        activate LoanRepository
        LoanRepository-->>LoanService: closedLoan
        deactivate LoanRepository

        LoanService->>ReaderRepository: getReaderEmail(loan.reader_id)
        activate ReaderRepository
        ReaderRepository-->>LoanService: readerEmail
        deactivate ReaderRepository

        opt Адресу електронної пошти знайдено: readerEmail != null
            LoanService->>EmailService: sendConfirmation(readerEmail, lostBookReceiptPayload)
            activate EmailService
            EmailService-->>LoanService: deliveryStatus (SENT)
            deactivate EmailService
        end

        LoanService-->>ReturnController: LostBookResult(CLOSED, compensation=500.00, bookStatus=LOST)
    else Оплату компенсації відхилено: paymentResponse.status == DECLINED
        LoanService-->>ReturnController: LostBookResult(PAYMENT_PENDING, unpaidFine=500.00, bookStatus=LOST)
    end

    deactivate LoanService
    ReturnController-->>Reader: LostBookResponse(status, compensation=500.00, bookStatus=LOST)
    deactivate ReturnController
```