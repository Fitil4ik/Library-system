# Вимоги у синтаксисі EARS

REQ-1: When a Reader requests to borrow a Book, while the Book status is AVAILABLE, the System shall create a Loan record and set the Book status to LOANED.

REQ-2: When a Loan record is created or closed, the System shall trigger the EmailService to send a notification to the Reader.

REQ-3: If a Reader returns a Book after the return_date, then the System shall calculate the fine_amount for the Loan.

REQ-4: While a Loan has a fine_amount greater than zero, the System shall initiate the payment process via the PaymentGateway.

REQ-5: When a Reader reports a Book as lost, the System shall set the Book status to LOST and calculate the replacement fine_amount.

# User stories

US-1: Як Читач, я хочу перевіряти статус книги перед видачею, щоб переконатися, що вона доступна (AVAILABLE).

US-2: Як Читач, я хочу отримувати email-сповіщення при видачі та поверненні книги, щоб завжди знати свої терміни (return_date) та поточний стан оренди.

US-3: Як Читач, я хочу мати можливість безпечно оплачувати штрафи за прострочення через платіжний шлюз, щоб закрити свою заборгованість перед бібліотекою.

# Сценарії Gherkin (Given/When/Then)

Сценарій 1: Успішна видача доступної книги та обробка сповіщення зовнішнім сервісом
Given книга з певним id має статус AVAILABLE
And Читач успішно ідентифікований у системі
When Читач ініціює видачу цієї книги
Then Система створює нову сутність Loan та встановлює статус книги як LOANED
And Система передає корисне навантаження (payload) для сповіщення до EmailService
And EmailService відправляє лист-підтвердження з деталями оренди на email Читача
And EmailService повертає Системі статус про успішну доставку повідомлення

Сценарій 2: Повернення книги із простроченням та оплата штрафу
Given існує відкритий запис Loan для Читача
And поточна дата перевищує встановлений return_date
When Читач ініціює повернення книги
Then Система розраховує fine_amount у сутності Loan
And Система ініціює сесію оплати та передає суму штрафу до PaymentGateway
And PaymentGateway автентифікує платіжні дані Читача та списує кошти
And PaymentGateway надсилає Системі callback із підтвердженням успішної транзакції
And Система оновлює запис Loan як оплачений та змінює статус книги

# Матриця трасування

| Вимога | Прецеденти (Use Cases) | Користувацькі історії (US) | Gherkin сценарії |
| --- | --- | --- | --- |
| **REQ-1** (Створення Loan) | UC_Borrow, UC_CheckStatus | US-1, US-2 | Сценарій 1 |
| **REQ-2** (Сповіщення) | UC_Borrow, UC_Return, UC_Notify, UC_PayFine | US-1, US-2, US-3 | Сценарій 1, Сценарій 2 |
| **REQ-3** (Розрахунок штрафу) | UC_Return, UC_CheckStatus | US-2, US-3 | Сценарій 2 |
| **REQ-4** (Оплата штрафу) | UC_Return, UC_PayFine, UC_Notify | US-2, US-3 | Сценарій 2 |
| **REQ-5** (Втрата книги) | *Не покрито* | *Не покрито* | *Не покрито* |