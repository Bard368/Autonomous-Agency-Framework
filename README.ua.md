<div align="right">
  <a href="README.md"><img src="https://img.shields.io/badge/English-d9d9d9?style=for-the-badge" alt="English"></a>
  <a href="README.ua.md"><img src="https://img.shields.io/badge/Українська-1f8fff?style=for-the-badge" alt="Українська"></a>
</div>

# Autonomous Agency Framework (AAF)

![n8n](https://img.shields.io/badge/Orchestrator-n8n-FF6C37) ![Dify](https://img.shields.io/badge/AI_Engine-Dify-000000) ![PostgreSQL](https://img.shields.io/badge/Database-Neon_Postgres-336791) ![Docker](https://img.shields.io/badge/Infra-Docker_/_Traefik-2496ED)

**Autonomous Agency Framework (AAF)** — це подійно-орієнтована інфраструктура для розгортання та оркестрації автономних LLM-агентів у реальному B2B-бізнесі.

Проєкт замінює класичні відділи (Sales, L1/L2 Support) на мережу інтегрованих ШІ-мікросервісів. Система здатна самостійно аналізувати вебсайти лідів, проводити нативну кваліфікацію, керувати календарями та автономно вирішувати технічні інциденти клієнтів (Zero-Touch Resolution) з автоматичною ескалацією інженерам через Trello та IP-телефонію.

## 🧠 Технологічний стек
* **Оркестрація:** n8n (Мікросервісна архітектура, Sub-workflows).
* **AI-рушій:** Dify (DeepSeek / Gemini API).
* **База даних:** Neon (Serverless PostgreSQL).
* **Інтеграції:** Kommo CRM, Trello, Twilio, Google Calendar, Telegram API.
* **Інфраструктура:** Docker, Traefik (Auto-SSL).

---

## 🏛 Архітектура та надійність (Enterprise Grade)
Система спроєктована з урахуванням високих навантажень і захисту від критичних збоїв:
* **Ідемпотентність:** Захист від дублювання при повторних вебхуках завдяки транзакційним перевіркам станів (State Checks) у PostgreSQL.
* **Race Condition Protection (Follower > Leader):** Асинхронні медіафайли (наприклад, альбоми в Telegram) перехоплюються, конвертуються в `base64` і збираються в єдиний Payload через механізм очікування лідера, виключаючи розриви контексту сесії.
* **Timezone Fault Tolerance:** Уся інфраструктура жорстко синхронізована за UTC, що усуває помилки парсингу `dbTime` між Node.js та БД під час розрахунків SLA.

---

## 📂 Маршрутизація та Мікросервіси (n8n Workflows)

Бізнес-логіка розділена на ізольовані воркери (Workers), які запускаються центральним API-шлюзом.

### 1. API Gateway (Головний Оркестратор)
Єдина точка входу. Валідує Payload, виконує SQL-верифікацію відправника (Співробітник, Лід, Активний Клієнт), агрегує файли та направляє дані у відповідний пайплайн.

![Main API Contract](./images/orchestrator.png)
*На скриншоті: Маршрутизація трафіку та механізм Follower > Leader для обробки медіафайлів.*

### 2. Autonomous Sales (SDR Pipeline)
Блок пресейлу. При отриманні посилання на сайт від нового ліда, система автономно аналізує вебресурс, готує Payload для ШІ та створює картку в Kommo CRM. Також реалізовано блок валідації даних (через вузол If): система перевіряє повноту зібраної інформації перед автоматичним формуванням документів (КП).

![SDR Pipeline](./images/sdr_pipeline.png)
*На скриншоті: Аналіз сайту ліда, підготовка Payload для ШІ, створення сутності в CRM та перевірка повноти даних для генерації документів.*

### 3. Tech Support & Operations (Support Pipeline)
Інтелектуальна система підтримки L1/L2. Працює з історією сесій, відрізняє нові звернення від відповідей у рамках поточних тикетів та керує ескалаціями.

![Support Entry](./images/support_entry.png)
*На скриншоті: Ініціалізація сесії — перевірка типу користувача та верифікація активного тикета.*

![Support Escalation](./images/support_escalation.png)
*На скриншоті: SLA & Escalation Protocol — категоризація інцидентів із миттєвим створенням карток у Trello та автоматичним автодозвоном інженеру через Twilio.*

### 4. Interactive Events (Callback Router)

![Caqllback](./images/callback.png)
A dedicated microservice for handling asynchronous events from messengers (button clicks, replies, and commands). 
* **Feature:** Processes the `/release` command from the engineering team's closed chat, parsing infrastructure credentials and securely saving them to the database for future TAM agent access.


---

## 🗄️ Архітектура Бази Даних (Neon Serverless)
Фреймворк спирається на реляційну модель даних із суворим управлінням станами. Нижче наведено спрощену ER-діаграму основних таблиць:

```mermaid
erDiagram
    leads_pipeline ||--o{ Ticket_messages : generates
    leads_pipeline {
        uuid id
        string contact_id
        string client_status "lead / onboarding / active"
        jsonb contract_data
        text ai_pain_point
    }
    support_tickets ||--o{ Ticket_messages : contains
    support_tickets {
        string ticket_id
        string status
        string category "BUG / ESCALATION / CR"
    }
    Client_credentials {
        string contact_id
        jsonb credentials
    }
