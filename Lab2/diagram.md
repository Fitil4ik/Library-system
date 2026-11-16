# Use Case діаграма (Library-system)

Нижче наведено оновлену діаграму прецедентів у нотації Mermaid, що відображає взаємодію акторів (`Reader`, `PaymentGateway`, `EmailService`) із функціоналом системи управління бібліотекою. Узагальнення (generalization) позначено суцільними стрілками (`-->`), а залежності `<<include>>` та `<<extend>>` — пунктирними (`-.->`).

```mermaid
flowchart LR
    %% Актори (людина та зовнішні системи)
    Reader(["👤 Reader<br/>(Читач)"])
    PaymentGateway(["⚙️ «system»<br/>PaymentGateway"])
    EmailService(["⚙️ «system»<br/>EmailService"])

    %% Межі системи
    subgraph LibrarySystem ["Library Management System"]
        direction TB

        %% Пошук та узагальнення
        UC_Search(["UC_Search<br/>Пошук книги"])
        UC_SearchByTitle(["UC_SearchByTitle<br/>Пошук за назвою"])
        UC_SearchByAuthor(["UC_SearchByAuthor<br/>Пошук за автором"])

        %% Видача книги
        UC_Borrow(["UC_Borrow<br/>Видача книги"])

        %% Повернення книги та пов'язані прецеденти
        UC_Return(["UC_Return<br/>Повернення книги"])
        UC_CheckLoanStatus(["UC_CheckLoanStatus<br/>Перевірка статусу позики"])
        UC_SendNotification(["UC_SendNotification<br/>Відправка сповіщення"])
        UC_PayFine(["UC_PayFine<br/>Оплата штрафу"])

        %% Зв'язки узагальнення (Generalization — суцільні стрілки)
        UC_SearchByTitle -->|generalization| UC_Search
        UC_SearchByAuthor -->|generalization| UC_Search

        %% Зв'язки <<include>> (пунктирні стрілки)
        UC_Return -.->|«include»| UC_CheckLoanStatus
        UC_Return -.->|«include»| UC_SendNotification

        %% Зв'язки <<extend>> (пунктирні стрілки)
        UC_PayFine -.->|«extend»<br/>[прострочення або втрата]| UC_Return
    end

    %% Асоціації акторів
    Reader --- UC_Search
    Reader --- UC_Borrow
    Reader --- UC_Return

    UC_PayFine --- PaymentGateway
    UC_SendNotification --- EmailService
```