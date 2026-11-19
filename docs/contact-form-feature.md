# Контактна форма: телефон, вкладення, сповіщення

Документ потрібен, щоб реалізація й приймання змін контактної форми спирались лише на GitHub-issue та перевірений код, без припущень. Невідоме винесено в відкриті питання.

Джерела: [GitHub issue #2](https://github.com/romaberezhok/sdd-workshop-roman-berezhok/issues/2), поточна реалізація форми/API/email, аналіз у цій розмові. Дані з grilling / `.scratch/.../spec.md` **не** використовуються.

## 1. Короткий опис і вихідний запит

Потрібно посилити контактну форму: зробити телефон обов’язковим, дозволити вкладення документа (CV або портфоліо) і відобразити нове поле в сповіщеннях.

Оригінал із [issue #2](https://github.com/romaberezhok/sdd-workshop-roman-berezhok/issues/2) (секція Original request) дослівно:

> We want to tighten up the contact form. Phone number should be mandatory.
> We should also let visitors attach a document (CV or portfolio) when they write
> to us. Notifications need to reflect the new field.

## 2. Бізнес-контекст

### Ціль

З issue #2:

- телефон обов’язковий;
- відвідувач може прикріпити документ (CV або portfolio);
- сповіщення мають відображати нове поле (вкладення).

### Метрика

Не задана в issue #2, розмові чи коді → **Q-16**.

### Обмеження

З коду / архітектури застосунку (не з issue):

- сайт навчальний; `/api/collaboration` зараз відповідає demo-успехом після валідації і **не** викликає `emailService`;
- UI на contacts/cooperation шле JSON на `/api/collaboration`; `/api/contact` у браузерному шляху не викликається;
- `ContactForm` також використовується в `GetNotesModal` — зміна спільного компонента зачіпає цю поверхню (як саме поводитись → **Q-06**, **Q-15**).

Інші обмеження (збереження файлу, SMTP, формати, розмір, GetNotes «без змін») у issue #2 **не** сформульовані → відкриті питання.

## 3. User stories

> Як відвідувач форми контактів / співпраці, я хочу обов’язково вказати телефон, щоб зі мною могли зв’язатися.

> Як відвідувач, я хочу за потреби додати документ (CV або portfolio), щоб одразу передати матеріали з запитом.

> Як отримувач сповіщення про запит, я хочу бачити в ньому нове поле (вкладення), щоб не губити контекст матеріалів.

## 4. Діаграма послідовності

Поточний шлях (як у коді сьогодні) і цільові зміни, підтверджені лише issue #2 (обов’язковий телефон, вкладення, сповіщення). Деталі транспорту/збереження — у Q.

```mermaid
sequenceDiagram
  actor Visitor as Відвідувач
  participant Form as ContactForm
  participant Offer as OfferCollaborationForm
  participant API as POST /api/collaboration
  participant Validate as validateContactData
  participant Mail as Email templates

  Visitor->>Form: name, email, phone, message, policy, attachment?
  Form->>Form: клієнтська валідація

  alt невалідно
    Form-->>Visitor: інлайн-помилки; запит не йде
  else валідно
    Form->>Offer: onSubmit
    Offer->>API: зараз JSON; майбутній формат — Q-14
    API->>Validate: поля (+ attachment, якщо вирішать на сервері)

    alt помилка валідації / API
      API-->>Offer: помилка
      Note over Offer,Form: UX при фейлі — Q-05
    else demo OK (поточна поведінка API)
      API-->>Offer: 200
      Note over Mail: шаблони мають відобразити нове поле (issue #2); виклик з API зараз відсутній — Q-09
      Form-->>Visitor: confirmation modal (поточна поведінка)
    end
  end
```

## 5. Таблиця правил

Три стани: **Є зараз** (код) · **Має бути** (лише issue #2 / явно з розмови про код) · **Хто вирішив**. Порожнє = не з’ясовано.

| ID | Правило | Є зараз | Має бути | Хто вирішив |
| --- | --- | --- | --- | --- |
| RULE-01 | Ім’я обов’язкове, 2–50 символів | так (`lengths`, zod) | | |
| RULE-04 | Email обов’язковий і валідний | так | | |
| RULE-07 | Телефон на формі запиту | опційний (`phoneSchema.optional()`, `phoneNumber?` у валідаторі) | обов’язковий | issue #2 |
| RULE-08 | Якщо телефон заповнений: `+` і цифри, довжина 11–16 | так (`regex.PHONE_*`, `lengths.PHONE_*`, маска) | | |
| RULE-09 | Помилка формату телефону на UI | uk/en `phoneNumberInvalid` | + помилка обов’язковості (текст → Q-03) | |
| RULE-10 | Помилка телефону на API | `errors.PHONE_NUMBER_ERROR`; відсутній телефон зараз OK | відхиляти відсутній (наслідок RULE-07) | issue #2 (логічний наслідок) |
| RULE-11 | Лейбл телефону без `*` | так | | |
| RULE-12 | Повідомлення 10–1000 символів | так | | |
| RULE-15 | Вкладення CV / portfolio | немає | дозволити прикріпити документ | issue #2 |
| RULE-16 | Тип, розмір, MIME, clear, кількість файлів | немає | | |
| RULE-17 | `policy` за замовчуванням `false` | так | | |
| RULE-18 | `policy` обов’язковий на UI | так (`policyRequired`) | | |
| RULE-19 | Текст згоди + лінк Privacy Policy | так | | |
| RULE-20 | `policy` не відправляється на API | так (`OfferCollaborationForm` шле name/email/phone/message) | | |
| RULE-21 | Після `onSubmit` — confirmation, потім reset / `policy=false` | так | | |
| RULE-22 | Транспорт тіла запиту | `application/json` | | |
| RULE-23 | `/api/collaboration` і `/api/contact` | спільний `validateContactData`, JSON, demo success | | |
| RULE-24 | Браузерний submit іде на `/api/collaboration` | так | | |
| RULE-25 | Телефон у шаблоні листа | лише якщо переданий | | |
| RULE-26 | Відображення вкладення в сповіщенні | немає | сповіщення відображають нове поле (вкладення) | issue #2 |
| RULE-27 | SMTP / персистенція файлу | API не викликає `emailService`; поля файлу немає | | |
| RULE-28 | `GetNotesModal` використовує той самий `ContactForm` | так | | |

## 6. Критерії приймання

### AC-01 — Порожній телефон блокує submit

- **Given** відвідувач на `/contacts` або `/cooperation` з порожнім телефоном і валідними іншими обов’язковими полями (включно з `policy`)
- **When** натискає «Надіслати запит» / «Send request»
- **Then** запит на API не йде; біля телефону інлайн-помилка обов’язковості мовою інтерфейсу (uk і en)
- **Rules:** RULE-07, RULE-09

### AC-02 — Невалідний телефон блокує submit

- **Given** телефон не порожній, але невалідний або неповний відносно поточних правил формату в коді
- **When** відвідувач надсилає форму
- **Then** запит не йде; показується помилка формату: uk «Перевірте формат номера телефону», en «Check the phone number format»
- **Rules:** RULE-08, RULE-09

### AC-03 — Успіх без файлу

- **Given** валідні обов’язкові поля з валідним телефоном, файл не вибрано (або поле ще не додане — після появи поля), `policy=true`
- **When** submit
- **Then** запит приймається в поточному demo-режимі API; confirmation modal показується; під час відправки кнопка `disabled`
- **Rules:** RULE-07, RULE-15, RULE-18, RULE-21

### AC-04 — Успіх з вкладенням

- **Given** валідні поля й прикріплений документ (CV або portfolio), що відповідає ще не затвердженим правилам файлу (Q-02, Q-07, RULE-16)
- **When** submit
- **Then** запит успішний за узгодженими правилами валідації файлу; сповіщення / шаблон може відобразити нове поле (AC-11)
- **Rules:** RULE-15, RULE-26

### AC-05 — Файл не відповідає правилам (розмір)

- **Given** вибрано файл, що перевищує узгоджений ліміт розміру (ліміт → Q-02)
- **When** спроба надіслати
- **Then** submit не виконується; інлайн-помилка uk і en
- **Rules:** RULE-16

### AC-06 — Файл не відповідає правилам (тип / MIME)

- **Given** файл поза узгодженим allowlist типів (Q-07)
- **When** спроба submit
- **Then** інлайн-помилка uk/en; за наявності серверної перевірки — запит не вважається успішним
- **Rules:** RULE-16

### AC-07 — Надсилання без вкладення після clear

- **Given** відвідувач вибрав файл і зняв його, якщо clear передбачено (Q-17)
- **When** надсилає форму без вкладення
- **Then** успіх як у AC-03 (вкладення опційне за issue #2)
- **Rules:** RULE-15

### AC-08 — Політика конфіденційності обов’язкова

- **Given** `policy=false` (дефолт) і валідні інші поля
- **When** submit
- **Then** uk «Щоб продовжити, потрібно дати згоду» / en «You must agree to continue»; запит не йде; лінк на Privacy Policy лишається
- **Rules:** RULE-17, RULE-18, RULE-19

### AC-09 — Policy не їде на API

- **Given** успішний клієнтський submit з `policy=true`
- **When** формується запит до API
- **Then** у payload немає `policy` (як зараз)
- **Rules:** RULE-20

### AC-10 — Стан кнопки під час відправки

- **Given** запит іде в польоті (`isSubmitting` у `OfferCollaborationForm`)
- **When** UI у цьому стані
- **Then** кнопка submit `disabled`
- **Rules:** поведінка `OfferCollaborationForm`

### AC-11 — Сповіщення відображають нове поле

- **Given** дані запиту з вкладенням (метадані або інший узгоджений спосіб — Q-09, Q-18)
- **When** формується сповіщення (шаблон листа)
- **Then** нове поле (вкладення) відображене у сповіщенні
- **Rules:** RULE-26

### AC-12 — Сповіщення без вкладення

- **Given** валідний запит без файлу
- **When** формується сповіщення
- **Then** немає відображення вкладення як наявного файлу (точний вигляд → Q-18)
- **Rules:** RULE-26

### AC-13 — Тексти помилок uk і en

- **Given** локаль `uk` або `en`
- **When** порожній/невалідний телефон, помилка файлу або знятий `policy`
- **Then** тексти з `i18n/messages/{uk|en}.json` для відомих ключів; нові ключі — узгоджені пари uk/en (Q-03)
- **Rules:** RULE-09, RULE-18

### AC-14 — Приватність і зберігання файлу

- **Given** запит з вкладенням
- **When** сервер обробив запит
- **Then** поведінка зберігання відповідає відповіді на Q-08 / Q-11 / Q-12 (поки не закриті — AC не вважається виконаним)
- **Rules:** RULE-27

### AC-15 — Вплив на GetNotes

- **Given** `GetNotesModal` рендерить `ContactForm`
- **When** змінюють обов’язковість телефону / додають поле файлу в `ContactForm`
- **Then** поведінка GetNotes відповідає рішенню Q-06 / Q-15 (не зафіксована в issue #2)
- **Rules:** RULE-28

### AC-16 — Серверна валідація телефону

- **Given** запит на `/api/collaboration` або `/api/contact` без телефону або з невалідним телефоном
- **When** сервер валідує через `validateContactData`
- **Then** запит не вважається успішним (наслідк обов’язкового телефону з issue #2)
- **Rules:** RULE-07, RULE-08, RULE-10

## 7. Таблиця ключових файлів

| Призначення | Шлях |
| --- | --- |
| UI форми | `app/shared/components/forms/contact-form/ContactForm.tsx` |
| Submit / fetch | `app/shared/components/forms/offer-collaboration-form/OfferCollaborationForm.tsx` |
| GetNotes (спільний ContactForm) | `app/shared/components/get-notes-modal/GetNotesModal.tsx` |
| Серверна валідація | `app/lib/utils/validateContactData.ts` |
| Константи довжин / regexp | `app/constants/validation.ts` |
| Повідомлення помилок API | `app/constants/errors.ts` |
| API | `app/api/collaboration/route.ts`, `app/api/contact/route.ts` |
| Email-шаблони | `app/services/email/emails/emails.ts` |
| Email stub-сервіс | `app/services/email/emailService.ts` |
| i18n | `i18n/messages/uk.json`, `i18n/messages/en.json` |
| Тести валідатора | `app/lib/utils/validateContactData.test.ts` |
| Тести форми | `app/shared/components/forms/contact-form/ContactForm.test.tsx` |
| Вихідний запит | [GitHub issue #2](https://github.com/romaberezhok/sdd-workshop-roman-berezhok/issues/2) |

## 8. Відкриті питання

| ID | Питання | Чому блокує | Хто відповідає | До дати | AC / RULE |
| --- | --- | --- | --- | --- | --- |
| Q-01 | Ліміти телефону 11–16 і повідомлення 1000 лишаються без змін? | Інакше роз’їдуться zod, `maxLength`, i18n і сервер | Product Owner + Tech Lead | — | AC-02; RULE-08, RULE-12 |
| Q-02 | Який максимальний розмір вкладення і в яких одиницях? | Немає в issue #2; блокує RULE-16 і AC-05 | PO | — | AC-04, AC-05; RULE-16 |
| Q-03 | Точні тексти uk/en для обов’язкового телефону та помилок файлу? | Немає в i18n і issue | Content / PO | — | AC-01, AC-13 |
| Q-04 | Серверні `errors.*` лишаються англійськими літералами? | Контракт 400 | Tech Lead + BA | — | AC-16; RULE-10 |
| Q-05 | При помилці API чи показувати confirmation (зараз помилка може ковтатись)? | Інакше хибний «успіх» | PO + Tech Lead | — | AC-03, AC-10; RULE-21 |
| Q-06 | Чи змінювати GetNotes разом із `ContactForm`, чи потрібен окремий режим? | Спільний компонент; issue #2 мовчить | PO + Tech Lead | — | AC-15; RULE-28 |
| Q-07 | Які формати файлів (extension/MIME) дозволені? | Немає в issue #2 | PO | — | AC-04, AC-06; RULE-16 |
| Q-08 | Де зберігається файл: диск, blob, лише метадані в листі, ніде? | Немає в issue #2 | PO + Tech Lead | — | AC-14; RULE-27 |
| Q-09 | Чи викликати генерацію/відправку листа з API в demo, чи лише оновити шаблони? | Зараз API не викликає `emailService` | PO + Tech Lead | — | AC-11; RULE-26, RULE-27 |
| Q-10 | Чи потрібен паритет `/api/contact` з `/api/collaboration`? | UI б’є лише в collaboration | Tech Lead | — | AC-16; RULE-23, RULE-24 |
| Q-11 | Чи достатньо наявного `policy` для прийому CV/portfolio? | Issue #2 не згадує окрему згоду | Legal / PO | — | AC-08, AC-14; RULE-19 |
| Q-12 | Термін зберігання файлу / метаданих / телефону? | Не описано | Legal + PO | — | AC-14 |
| Q-13 | Breaking change для запитів без телефону на API — ОКково? | Поточні тести дозволяють без телефону | PO | — | AC-16; RULE-07 |
| Q-14 | Залишаємо JSON чи переходимо на multipart через файл? | Файл у JSON незручний; issue #2 мовчить | Tech Lead | — | RULE-22 |
| Q-15 | Обов’язковий телефон у GetNotes — так чи ні? | Наслідок RULE-07 + спільного UI | PO | — | AC-15; RULE-07, RULE-28 |
| Q-16 | Яка метрика успіху фічі? | Немає в issue #2 | PO | — | бізнес-контекст |
| Q-17 | Чи потрібен clear вибраного файлу до submit? | Немає в issue #2 | PO | — | AC-07 |
| Q-18 | Що саме показати в сповіщенні про вкладення (ім’я, розмір, посилання, binary)? | Issue каже лише «reflect the new field» | PO | — | AC-11, AC-12; RULE-26 |
| Q-19 | Чи має телефон завжди з’являтись у тілі листа, чи достатньо лише поля вкладення? | Issue: «the new field» (вкладення); телефон у листі окремо не описаний | PO | — | AC-11; RULE-25, RULE-26 |

Дати відповідей **не призначені** (стовпець «До дати» = —).
