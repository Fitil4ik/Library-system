# Вимоги у синтаксисі EARS

REQ-1: When a Reader requests to borrow a Book, while the Book status is AVAILABLE, the System shall create a Loan record and set the Book status to LOANED.

REQ-2: When a Loan record is created or closed, the System shall trigger the EmailService to send a notification to the Reader.

REQ-3: If a Reader returns a Book after the return_date, then the System shall calculate the fine_amount for the Loan.

REQ-4: While a Loan has a fine_amount greater than zero, the System shall initiate the payment process via the PaymentGateway.

# User stories

US-1: Як Читач, я хочу перевіряти статус книги перед видачею, щоб переконатися, що вона доступна (AVAILABLE).

US-2: Як Читач, я хочу отримувати email-сповіщення при видачі та поверненні книги, щоб завжди знати свої терміни (return_date) та поточний стан оренди.

US-3: Як Читач, я хочу мати можливість безпечно оплачувати штрафи за прострочення через платіжний шлюз, щоб закрити свою заборгованість перед бібліотекою.

# Сценарії Gherkin (Given/When/Then)

Сценарій 1: Успішна видача доступної книги
Given книга з певним id має статус AVAILABLE
And Читач успішно ідентифікований за email
When Читач ініціює видачу цієї книги
Then Система створює нову сутність Loan із прив'язкою reader_id та book_id
And Система встановлює статус книги як LOANED
And Система викликає EmailService для відправки підтвердження

Сценарій 2: Повернення книги із простроченням та оплата штрафу
Given існує запис Loan для Читача
And поточна дата перевищує встановлений return_date
When Читач ініціює повернення книги
Then Система розраховує fine_amount
And Система пропонує оплату штрафу через PaymentGateway
And Система оновлює статус книги після завершення процесу

# Матриця трасування

| Вимога | Прецеденти (Use Cases) | Користувацька історія | Gherkin сценарій |
| --- | --- | --- | --- |
| REQ-1 | UC_Borrow, UC_CheckStatus | US-1 | Сценарій 1 |
| REQ-2 | UC_Borrow, UC_Return, UC_Notify | US-2 | Сценарій 1 |
| REQ-3 | UC_Return | US-3 | Сценарій 2 |
| REQ-4 | UC_Return, UC_PayFine | US-3 | Сценарій 2 |