# Техническая спецификация

| Сценарий | Описание | Ключевые функции |
|----------|----------|------------------|
| **Lecture-Free** | AI-тьютор для самостоятельного изучения | Персонализированные лонгриды, граф знаний, диалог с компаньоном, диагностики понимания |
| **Кейс-тренажер** | Практико-ориентированное обучение через диалоговые кейсы | Симуляция бизнес-ситуаций, пошаговый диалог, оценка решений с feedback |
| **AI-помощник** | Универсальный ассистент для навигации по курсу | RAG по материалам курса, ответы на вопросы, рекомендации по изучению |

- **Масштаб**: до 2000 активных студентов
- **Инфраструктура**: Yandex Cloud
- **Архитектура**: микросервисная

---

## 2. Архитектура системы

### 2.1 Принципы проектирования

**Почему микросервисы:**
- **Независимое масштабирование**: AI-сервисы требуют больше ресурсов при пиковых нагрузках, чем CRUD-операции
- **Изоляция отказов**: сбой в генерации контента не блокирует базовую работу платформы
- **Технологическая гибкость**: ML-компоненты на Python, real-time на Node.js
- **Параллельная разработка**: разработка идет независимо над разными сервисами

**Почему Event-Driven для части коммуникаций:**
- Отслеживание прогресса студентов требует audit trail всех действий
- Асинхронная генерация контента не должна блокировать UI
- ML-пайплайны (персонализация, оценка) работают в фоне

### 2.2 Высокоуровневая архитектура

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                        │
│         Web App (React)              Mobile (PWA)           Author Tool     │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           YANDEX API GATEWAY                                │
│                    (Routing, Rate Limiting, JWT Auth)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         ▼                             ▼                             ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────────┐
│   AUTH SERVICE  │         │   CORE SERVICE  │         │    AI GATEWAY       │
│                 │         │                 │         │                     │
│  • Регистрация  │         │  • Курсы        │         │  • LLM Routing      │
│  • JWT tokens   │         │  • Материалы    │         │  • Rate Limiting    │
│  • Роли (RBAC)  │         │  • Группы       │         │  • Response Cache   │
│  • Профили      │         │  • Доступ       │         │  • Streaming (SSE)  │
└─────────────────┘         └─────────────────┘         └─────────────────────┘
                                       │                             │
                                       │         ┌───────────────────┼───────────────────┐
                                       │         ▼                   ▼                   ▼
                                       │  ┌─────────────┐   ┌─────────────────┐   ┌───────────────┐
                                       │  │ AI COMPANION│   │ KNOWLEDGE GRAPH │   │PERSONALIZATION│
                                       │  │   SERVICE   │   │    SERVICE      │   │    SERVICE    │
                                       │  │             │   │                 │   │               │
                                       │  │ • RAG       │   │ • Генерация     │   │ • Колб-тест   │
                                       │  │ • Диалог    │   │ • Редактирование│   │ • Интересы    │
                                       │  │ • Контекст  │   │ • Маршрут       │   │ • Адаптация   │
                                       │  └─────────────┘   └─────────────────┘   └───────────────┘
                                       │                                                  │
         ┌─────────────────────────────┼──────────────────────────────────────────────────┘
         ▼                             ▼                             ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────────┐
│   ASSESSMENT    │         │    CONTENT      │         │     ANALYTICS       │
│    SERVICE      │         │    SERVICE      │         │      SERVICE        │
│                 │         │                 │         │                     │
│  • Диагностики  │         │  • Загрузка     │         │  • Статус-матрица   │
│  • Кейсы        │         │  • Парсинг      │         │  • Прогресс         │
│  • Оценка       │         │  • Чанкинг      │         │  • Вопросы студентов│
│  • Результаты   │         │  • Хранение     │         │  • Портфолио        │
└─────────────────┘         └─────────────────┘         └─────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │  PostgreSQL  │  │    Redis     │  │    Qdrant    │  │  Object Storage  │ │
│  │  (основные   │  │  (кэш,       │  │  (векторы,   │  │  (файлы,         │ │
│  │   данные)    │  │   сессии)    │  │   embeddings)│  │   медиа)         │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EVENT BUS (Kafka)                                 │
│              (Прогресс студентов, Фоновые задачи, ML-пайплайны)             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Микросервисы — детальное описание

#### 2.3.1 Auth Service

**Назначение**: Аутентификация, авторизация и управление пользователями.

**Функции**:
- Регистрация и аутентификация (email + password, OAuth)
- Выпуск и валидация JWT токенов (access + refresh)
- Ролевая модель: `author`, `student`, `admin`
- Хранение профилей пользователей

**Почему отдельный сервис**: 
- Критичность для безопасности требует изоляции
- Возможность использования специализированных решений (Keycloak, Supertokens)
- Независимое масштабирование при пиках регистраций

**Технологии**: FastAPI, PostgreSQL, Redis (сессии)

**API**:
```
POST /auth/register         # Регистрация
POST /auth/login            # Вход, получение токенов
POST /auth/refresh          # Обновление access token
GET  /auth/me               # Текущий пользователь
PUT  /users/{id}/profile    # Обновление профиля
```

---

#### 2.3.2 Core Service

**Назначение**: Основная бизнес-логика платформы — курсы, материалы, группы.

**Функции**:
- CRUD операции для модулей/курсов
- Управление компетенциями и образовательными результатами
- Группы студентов и ссылки доступа
- Статусы публикации (draft → published → archived)

**Почему отдельный сервис**:
- Центральная точка бизнес-логики
- Стабильная нагрузка, предсказуемое масштабирование
- Высокие требования к консистентности данных

**Технологии**: FastAPI, PostgreSQL, SQLAlchemy

**API**:
```
# Модули
POST   /modules                     # Создание модуля
GET    /modules/{id}                # Получение модуля
PUT    /modules/{id}                # Редактирование
POST   /modules/{id}/publish        # Публикация
POST   /modules/{id}/access-link    # Генерация ссылки доступа

# Компетенции
POST   /modules/{id}/competencies   # Добавление компетенций
GET    /modules/{id}/competencies   # Список компетенций
POST   /modules/{id}/outcomes       # Образовательные результаты

# Группы
POST   /groups                      # Создание группы
GET    /modules/{id}/students       # Студенты модуля
POST   /groups/{id}/join            # Присоединение по ссылке
```

---

#### 2.3.3 Content Service

**Назначение**: Загрузка, обработка и хранение учебных материалов.

**Функции**:
- Загрузка файлов (PDF, DOCX, TXT, видео, изображения)
- Парсинг документов с сохранением структуры
- Чанкирование текста для RAG
- Хранение дополнительного контента (видео, код, таблицы)

**Почему отдельный сервис**:
- Ресурсоёмкие операции парсинга не должны влиять на основной API
- Асинхронная обработка больших файлов
- Интеграция с Object Storage

**Технологии**: FastAPI, MinIO/S3, PyMuPDF, python-docx

**API**:
```
POST   /modules/{id}/files              # Загрузка файлов
POST   /modules/{id}/files/url          # Загрузка по URL
GET    /modules/{id}/files              # Список файлов
DELETE /files/{id}                      # Удаление файла
POST   /modules/{id}/syllabus           # Загрузка РПД
GET    /sections/{id}/source            # Исходные чанки раздела
POST   /sections/{id}/additional        # Добавление доп. контента
```

---

#### 2.3.4 Knowledge Graph Service

**Назначение**: Генерация, редактирование и навигация по графу знаний.

**Функции**:
- Автоматическая генерация графа из загруженных материалов
- Извлечение концептов и определение связей (prerequisites, relates_to)
- Сопоставление с учебным планом (РПД)
- Редактирование графа автором
- Генерация рекомендованного маршрута изучения

**Почему отдельный сервис**:
- Специфическая логика работы с графовыми структурами
- Интенсивное использование LLM для извлечения концептов
- Независимое кэширование графов

**Технологии**: FastAPI, PostgreSQL + Apache AGE (графовое расширение), NetworkX

**Алгоритм генерации графа (методика ВШЭ)**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Чанки     │───▶│ Кластеризация│───▶│  Концепты  │
│  документов │    │  (HDBSCAN)  │    │  (LLM)      │
└─────────────┘    └─────────────┘    └─────────────┘
                                             │
                         ┌───────────────────┴───────────────────┐
                         ▼                                       ▼
                  ┌─────────────┐                         ┌─────────────┐
                  │Сопоставление│                         │ Определение │
                  │   с РПД     │                         │   связей    │
                  └─────────────┘                         └─────────────┘
                         │                                       │
                         └───────────────────┬───────────────────┘
                                             ▼
                                      ┌─────────────┐
                                      │  Маршрут    │
                                      │(топ. сорт.) │
                                      └─────────────┘
```

**API**:
```
POST   /modules/{id}/graph/generate     # Запуск генерации
GET    /modules/{id}/graph/status       # Статус генерации
GET    /modules/{id}/graph              # Получение графа
PUT    /modules/{id}/graph/concepts     # Редактирование концептов
PUT    /modules/{id}/graph/relations    # Редактирование связей
POST   /modules/{id}/graph/confirm      # Подтверждение автором
GET    /modules/{id}/graph/route        # Рекомендованный маршрут
POST   /graph/edit-prompt               # Редактирование через LLM
```

---

#### 2.3.5 AI Gateway

**Назначение**: Единая точка входа для всех LLM-запросов с маршрутизацией и кэшированием.

**Функции**:
- Маршрутизация запросов между моделями (YandexGPT, GigaChat)
- Rate limiting per user/module
- Семантическое кэширование частых запросов
- Streaming ответов через SSE
- Retry logic и fallback при сбоях

**Почему отдельный сервис**:
- Централизованный контроль затрат на LLM
- Единая точка мониторинга качества ответов
- Возможность A/B тестирования моделей
- Изоляция от сбоев LLM провайдеров

**Технологии**: FastAPI, Redis (кэш), SSE streaming

**Паттерн маршрутизации**:
```
┌─────────────┐     ┌─────────────┐     ┌─────────────────────────────┐
│   Request   │────▶│  Semantic   │────▶│  Cache Hit?                 │
│             │     │   Cache     │     │  Yes → Return cached        │
└─────────────┘     └─────────────┘     │  No  → Route to LLM         │
                                        └─────────────────────────────┘
                                                      │
                                        ┌─────────────┴─────────────┐
                                        ▼                           ▼
                                 ┌─────────────┐             ┌─────────────┐
                                 │ GPT_model   │             │  Alt_model  │
                                 │ (primary)   │             │ (fallback)  │
                                 └─────────────┘             └─────────────┘
```

**API**:
```
POST   /llm/generate                    # Генерация ответа
POST   /llm/stream                      # Streaming генерация (SSE)
POST   /llm/embed                       # Получение embeddings
GET    /llm/models                      # Доступные модели
```

---

#### 2.3.6 AI Companion Service

**Назначение**: Диалоговый AI-компаньон с RAG по материалам курса.

**Функции**:
- Ответы на вопросы студентов с использованием RAG
- Поддержание контекста диалога
- Контекст текущей страницы (объяснение выделенного текста)
- Режимы: "объясни проще", "приведи пример по интересам"
- Сократический метод (наводящие вопросы)

**Почему отдельный сервис**:
- Сложная логика управления контекстом диалога
- Интеграция с несколькими источниками данных (Vector DB, Graph, Profile)
- Streaming ответов требует persistent connections

**Технологии**: FastAPI, LangChain, Qdrant, WebSocket/SSE

**RAG Pipeline**:
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RAG PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │  Query   │──▶│  Embed   │──▶│  Hybrid  │──▶│  Rerank  │──▶│ Context  │   │
│  │          │   │          │   │  Search  │   │          │   │ Assembly │   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘   │
│                                     │                              │        │
│                    ┌────────────────┴────────────────┐             │        │
│                    ▼                                ▼              ▼        │
│             ┌──────────┐                     ┌──────────┐   ┌──────────┐    │
│             │  Vector  │                     │  Graph   │   │   LLM    │    │
│             │  Search  │                     │  Lookup  │   │ Generate │    │
│             │ (Qdrant) │                     │  (AGE)   │   │          │    │
│             └──────────┘                     └──────────┘   └──────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Структура контекста диалога**:
```
System Prompt:
├── Base persona (роль AI-тьютора)
├── Course context (название, цели курса)
├── Student profile (стиль Колба, интересы, уровень)
├── Page context (текущий раздел, если есть)
├── RAG context (релевантные чанки, top-5)
├── Graph context (связанные концепты)
└── Conversation history (последние 20 сообщений)
```

**API**:
```
POST   /chat/message                    # Отправка сообщения
WS     /chat/stream                     # WebSocket для streaming
POST   /chat/context                    # Установка контекста страницы
GET    /chat/history/{module_id}        # История диалогов
POST   /content/simplify                # "Объясни проще"
POST   /content/example                 # "Приведи пример"
```

---

#### 2.3.7 Personalization Service

**Назначение**: Персонализация контента под профиль студента.

**Функции**:
- Тест Колба для определения стиля обучения
- Сбор интересов студента (минимум 3)
- Выбор стиля компаньона (формальный, неформальный, кастомный)
- Генерация персонализированного контента
- Адаптация сложности материала

**Почему отдельный сервис**:
- Отдельная доменная область со своей логикой
- Ресурсоёмкая генерация персонализированного контента
- Кэширование для разных комбинаций профилей

**Технологии**: FastAPI, PostgreSQL, Redis (кэш)

**Стили Колба и адаптация контента**:

| Стиль | Характеристики | Адаптация контента |
|-------|----------------|-------------------|
| **Diverger** | Наблюдение, генерация идей | Примеры из жизни, визуализация, разные точки зрения |
| **Assimilator** | Логика, теория | Структурированная информация, схемы, классификации |
| **Converger** | Практика, решение задач | Алгоритмы, пошаговые инструкции, практические задания |
| **Accommodator** | Эксперименты, действия | Интерактивные элементы, вопросы для размышления |

**Структура персонализированного лонгрида**:
```
1. Заголовок
2. Суммаризация (3-4 тезиса, адаптация по сложности)
3. Ключевые понятия [Понятие — описание]
4. Основной текст (персонализация по Колбу и сложности)
5. Примеры (персонализация по интересам студента)
6. Выжимка содержания
7. Дополнительный контент автора (без изменений)
8. Связанные концепты (из графа)
9. Кнопка перехода к диагностике
```

**API**:
```
GET    /kolb/questions                  # Вопросы теста Колба
POST   /kolb/submit                     # Отправка ответов
PUT    /profile/interests               # Обновление интересов
PUT    /profile/companion-style         # Выбор стиля компаньона
PUT    /profile/complexity              # Уровень сложности
POST   /modules/{id}/personalize        # Генерация персонализированного контента
GET    /sections/{id}/personalized      # Получение персонализированного раздела
```

---

#### 2.3.8 Assessment Service

**Назначение**: Диагностики понимания и кейсовые задания.

**Функции**:
- Четыре типа диагностик: single choice, multiple choice, matching, input
- Автогенерация вопросов по контенту
- Кейсовые диагностики с диалоговой оценкой
- Оценка ответов через LLM (для открытых вопросов и кейсов)
- Хранение результатов и статистики

**Почему отдельный сервис**:
- Сложная логика оценки требует изоляции
- Независимое масштабирование при массовых тестированиях
- Специфические требования к хранению результатов

**Технологии**: FastAPI, PostgreSQL, LangChain (для оценки)

**Кейсовая диагностика (методика ВШЭ)**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CASE DIAGNOSTIC FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────────┐                                                         │
│   │ Описание      │  Контекст бизнес-ситуации + Роль студента               │
│   │ ситуации      │                                                         │
│   └───────┬───────┘                                                         │
│           ▼                                                                 │
│   ┌───────────────┐                                                         │
│   │   Вопрос 1    │◀─────────────────────────────────────────┐              │
│   └───────┬───────┘                                          │              │
│           ▼                                                  │              │
│   ┌───────────────┐      ┌───────────────┐                   │              │
│   │ Ответ студента│─────▶│ LLM Judge     │                   │              │
│   └───────────────┘      │ (score 0-1)   │                   │              │
│                          └───────┬───────┘                   │              │
│                                  │                           │              │
│                    ┌─────────────┴─────────────┐             │              │
│                    ▼                           ▼             │              │
│            score >= 0.5              score < 0.5             │              │
│            ┌───────────┐            ┌───────────┐            │              │
│            │ Следующий │            │ Подсказка │────────────┘              │
│            │  вопрос   │            │ (max 2)   │       (retry)             │
│            └───────────┘            └───────────┘                           │
│                    │                                                        │
│                    ▼                                                        │
│            ┌───────────────────────────────────────┐                        │
│            │         ФИНАЛЬНЫЙ FEEDBACK            │                        │
│            │  • Оценка 1-5                         │                        │
│            │  • Пояснения по каждому вопросу       │                        │
│            │  • Рекомендации по изучению           │                        │
│            └───────────────────────────────────────┘                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**API**:
```
# Диагностики понимания
POST   /sections/{id}/diagnostics       # Создание диагностики
POST   /diagnostics/generate            # Автогенерация вопросов
GET    /diagnostics/{id}                # Получение диагностики
POST   /diagnostics/{id}/submit         # Отправка ответов

# Кейсовые диагностики
POST   /concepts/{id}/case              # Создание кейса
POST   /case/generate                   # Автогенерация кейса
POST   /case/{id}/start                 # Начало прохождения
POST   /case/{id}/answer                # Ответ на вопрос кейса
GET    /case/{id}/result                # Результат кейса

# Результаты
GET    /students/{id}/results           # Результаты студента
GET    /modules/{id}/results            # Результаты по модулю
```

---

#### 2.3.9 Analytics Service

**Назначение**: Аналитика для авторов и статистика для студентов.

**Функции**:
- Статус-матрица: студенты × концепты с прогрессом
- Детальная информация по студенту
- Суммаризация вопросов к компаньону (что спрашивают чаще всего)
- Портфолио студента (пройденные курсы, компетенции)
- Агрегированные метрики модуля

**Почему отдельный сервис**:
- Read-heavy нагрузка, отдельное масштабирование
- Сложные агрегации не должны нагружать основную БД
- Возможность использования специализированных хранилищ (ClickHouse)

**Технологии**: FastAPI, PostgreSQL (read replicas), Redis (кэш агрегатов)

**Статус-матрица**:
```
┌─────────────────────────────────────────────────────────────────┐
│                     СТАТУС-МАТРИЦА                              │
├─────────────────────────────────────────────────────────────────┤
│              │ Концепт 1 │ Концепт 2 │ Концепт 3 │ Концепт 4 │  │
├──────────────┼───────────┼───────────┼───────────┼───────────┼──┤
│ Студент А    │    ✓      │    ✓      │    ◐      │    ○      │  │
│ Студент Б    │    ✓      │    ◐      │    ○      │    ○      │  │
│ Студент В    │    ✓      │    ✓      │    ✓      │    ◐      │  │
└─────────────────────────────────────────────────────────────────┘
  ✓ = completed    ◐ = in progress    ○ = not started
```

**API**:
```
GET    /modules/{id}/status-matrix              # Статус-матрица
GET    /modules/{id}/students/{sid}/details     # Детали студента
GET    /modules/{id}/students/{sid}/questions   # Суммаризация вопросов
GET    /students/{id}/portfolio                 # Портфолио студента
GET    /modules/{id}/analytics                  # Агрегированные метрики
```

---

#### 2.3.10 Notification Service

**Назначение**: Уведомления и фоновые задачи.

**Функции**:
- Email-уведомления (welcome, напоминания, результаты)
- Push-уведомления (PWA)
- Обработка событий из Kafka
- Очередь фоновых задач (генерация, экспорт)

**Почему отдельный сервис**:
- Асинхронные операции не должны блокировать основные API
- Единая точка интеграции с внешними сервисами (email, push)
- Retry logic для ненадёжных операций

**Технологии**: Node.js (NestJS), Redis (Bull queue), Kafka consumer

---

## 3. Технологический стек

### 3.1 Backend

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| **API Framework** | FastAPI (Python) | Нативная async поддержка, автоматическая OpenAPI документация, отличная интеграция с ML-экосистемой (LangChain, NumPy, PyTorch). Типизация через Pydantic снижает количество ошибок |
| **Real-time сервис** | NestJS (Node.js) | Лучшая производительность для WebSocket/SSE, TypeScript из коробки, модульная архитектура для Notification Service |
| **ORM** | SQLAlchemy 2.0 | Зрелая библиотека с async поддержкой, Alembic для миграций, отличная интеграция с FastAPI |
| **Task Queue** | Celery + Redis | Проверенное решение для Python, поддержка периодических задач, retry logic |
| **LLM Orchestration** | LangChain + LangGraph | Абстракции для RAG, chains, agents; LangGraph для сложных диалоговых сценариев с состоянием |

### 3.2 Базы данных

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| **Primary DB** | PostgreSQL 15 + JSONB | ACID-гарантии для критичных данных, JSONB для гибких схем (профили, результаты), отличная производительность, managed-решение в Yandex Cloud |
| **Graph Extension** | Apache AGE | Расширение PostgreSQL для графов, Cypher-запросы, не требует отдельной инфраструктуры (vs Neo4j). Достаточно для графов до 100К узлов |
| **Vector DB** | Qdrant | Лучшая производительность среди open-source решений (~2ms latency), отличный metadata filtering, hybrid search. Self-hosted даёт полный контроль |
| **Cache** | Redis | Сессии, кэш LLM ответов, семантический кэш, очереди задач. Managed-решение в Yandex Cloud |
| **Object Storage** | Yandex Object Storage (S3) | Учебные материалы, медиа. Нативная интеграция с CDN, низкая стоимость |

### 3.3 Frontend

| Компонент | Технология | Обоснование |
|-----------|------------|-------------|
| **Framework** | React 18 + TypeScript | Крупнейшая экосистема, отличные библиотеки для EdTech (графы, редакторы). Типизация снижает ошибки |
| **Build Tool** | Vite | Мгновенный HMR, простая конфигурация, оптимизированные production-билды |
| **State Management** | TanStack Query (React Query) | Лучший кэш для API, оптимистичные обновления, автоматическая инвалидация. Zustand для UI-состояния |
| **UI Kit** | shadcn/ui + Tailwind | Копируемые компоненты (полный контроль), accessibility из коробки, быстрая кастомизация |
| **Graph Visualization** | React Flow | Лучшая интеграция с React, drag-n-drop из коробки, customizable nodes/edges |
| **Rich Text Editor** | TipTap | Расширяемая архитектура, collaborative editing ready, markdown support |
| **Real-time** | SSE / WebSocket | SSE для streaming LLM ответов (проще, reconnect из коробки), WebSocket для чата |

### 3.4 LLM

| Компонент | Технология                                     | Обоснование |
|-----------|------------------------------------------------|-------------|
| **Primary Model** | (YandexGPT Pro) но желательно продукты OpenAI) | Нативная интеграция с Yandex Cloud, оптимизация для русского языка, API совместим с OpenAI |
| **Fallback Model** | (GigaChat) Claude                              | Альтернатива при недоступности YandexGPT, хорошие результаты на математических задачах |
| **Embeddings** | YandexGPT Embeddings                           | Единая экосистема, 1024-мерные векторы, хорошее качество для русского текста |
| **Semantic Cache** | GPTCache + Redis                               | Снижение затрат на повторяющиеся запросы, threshold-based matching |

### 3.5 Инфраструктура (Yandex Cloud)

| Компонент | Сервис | Обоснование |
|-----------|--------|-------------|
| **Compute** | Managed Kubernetes | Автоматическое масштабирование, managed control plane, интеграция с другими сервисами |
| **API Gateway** | Yandex API Gateway | Serverless, OpenAPI 3.0, JWT authorizer, WebSocket support, интеграция с Cloud Functions |
| **Database** | Managed PostgreSQL | HA из коробки, автоматические бэкапы, мониторинг |
| **Cache** | Managed Redis | HA, persistence, managed upgrades |
| **Storage** | Object Storage + CDN | Учебные материалы, медиа с CDN для быстрой доставки |
| **Events** | Yandex Data Streams | Kafka API, managed, для event sourcing прогресса |
| **Monitoring** | Yandex Monitoring + DataLens | Метрики, логи, дашборды. DataLens для аналитических отчётов |

---

## 4. Модель данных

### 4.1 PostgreSQL — основные сущности

```sql
-- Пользователи
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    role VARCHAR(20) NOT NULL CHECK (role IN ('author', 'student', 'admin')),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Профиль студента (персонализация)
CREATE TABLE student_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    kolb_style VARCHAR(20) CHECK (kolb_style IN ('diverger', 'assimilator', 'converger', 'accommodator')),
    kolb_answers JSONB,
    interests JSONB NOT NULL DEFAULT '[]',
    complexity_level VARCHAR(20) DEFAULT 'intermediate',
    companion_style VARCHAR(20) DEFAULT 'informal',
    companion_custom_prompt TEXT,
    UNIQUE(user_id)
);

-- Модуль (курс)
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    scenario_type VARCHAR(20) NOT NULL CHECK (scenario_type IN ('lecture_free', 'case_trainer', 'ai_assistant')),
    author_id UUID REFERENCES users(id),
    status VARCHAR(20) DEFAULT 'draft',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Концепт (узел графа знаний)
CREATE TABLE concepts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    order_index INTEGER NOT NULL,
    from_syllabus BOOLEAN DEFAULT false
);

-- Раздел (подтема внутри концепта)
CREATE TABLE sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    concept_id UUID REFERENCES concepts(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    order_index INTEGER NOT NULL,
    content_essence JSONB  -- базовый контент для персонализации
);

-- Связи между концептами
CREATE TABLE concept_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_concept_id UUID REFERENCES concepts(id) ON DELETE CASCADE,
    target_concept_id UUID REFERENCES concepts(id) ON DELETE CASCADE,
    edge_type VARCHAR(20) NOT NULL CHECK (edge_type IN ('prerequisite', 'relates_to', 'next')),
    UNIQUE(source_concept_id, target_concept_id, edge_type)
);

-- Прогресс студента
CREATE TABLE student_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    concept_statuses JSONB DEFAULT '{}',
    current_concept_id UUID REFERENCES concepts(id),
    started_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, module_id)
);

-- Диагностики
CREATE TABLE diagnostics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id UUID REFERENCES sections(id) ON DELETE CASCADE,
    diagnostic_type VARCHAR(20) NOT NULL,
    question TEXT NOT NULL,
    options JSONB,
    correct_answer JSONB NOT NULL
);

-- Кейсовые диагностики
CREATE TABLE case_diagnostics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    concept_id UUID REFERENCES concepts(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    context_description TEXT NOT NULL,
    student_role TEXT NOT NULL,
    questions JSONB NOT NULL  -- массив вопросов с эталонами
);

-- Результаты диагностик
CREATE TABLE diagnostic_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    diagnostic_id UUID REFERENCES diagnostics(id) ON DELETE CASCADE,
    answer JSONB NOT NULL,
    is_correct BOOLEAN NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.2 Qdrant — векторные коллекции

```python
# Коллекция чанков документов
chunks_collection = {
    "name": "course_chunks",
    "vectors": {
        "size": 1024,  # YandexGPT Embeddings
        "distance": "Cosine"
    },
    "payload_schema": {
        "module_id": "keyword",
        "concept_id": "keyword",
        "section_id": "keyword",
        "source_file": "text",
        "text": "text"
    }
}

# Коллекция концептов (для semantic search)
concepts_collection = {
    "name": "course_concepts",
    "vectors": {
        "size": 1024,
        "distance": "Cosine"
    },
    "payload_schema": {
        "module_id": "keyword",
        "concept_id": "keyword",
        "title": "text",
        "description": "text"
    }
}
```

### 4.3 Redis — структуры данных

```
# Сессии пользователей
session:{session_id} → {user_id, role, expires_at}
TTL: 7 days

# Кэш персонализированного контента
personalized:{section_id}:{profile_hash} → {content_json}
TTL: 24 hours

# Семантический кэш LLM ответов
llm_cache:{query_hash}:{module_id} → {response_json}
TTL: 1 hour

# Rate limiting
rate_limit:{user_id}:ai_requests → counter
TTL: 1 minute

# Прогресс генерации
generation:{module_id}:graph → {status, progress_percent, error}
TTL: 1 hour
```

---

## 5. Схемы взаимодействия

### 5.1 Авторский флоу: Создание курса

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AUTHOR FLOW: Course Creation                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Author                                                                     │
│    │                                                                        │
│    │  1. Create Module                                                      │
│    ├────────────────────────▶ Core Service                                  │
│    │                              │                                         │
│    │  2. Upload Files             │                                         │
│    ├────────────────────────▶ Content Service                               │
│    │                              │                                         │
│    │                              │  2.1 Parse & Chunk                      │
│    │                              ├────────────────────▶ (internal)         │
│    │                              │                                         │
│    │                              │  2.2 Store Embeddings                   │
│    │                              ├────────────────────▶ Qdrant             │
│    │                              │                                         │
│    │                              │  2.3 Store Files                        │
│    │                              ├────────────────────▶ Object Storage     │
│    │                              │                                         │
│    │  3. Generate Graph           │                                         │
│    ├────────────────────────▶ KG Service                                    │
│    │                              │                                         │
│    │                              │  3.1 Extract Concepts                   │
│    │                              ├────────────────────▶ AI Gateway ──▶ LLM │
│    │                              │                                         │
│    │                              │  3.2 Build Relations                    │
│    │                              ├────────────────────▶ AI Gateway ──▶ LLM │
│    │                              │                                         │
│    │  4. Edit Graph               │                                         │
│    ├────────────────────────▶ KG Service                                    │
│    │                              │                                         │
│    │  5. Create Diagnostics       │                                         │
│    ├────────────────────────▶ Assessment Service                            │
│    │                              │                                         │
│    │  6. Publish                  │                                         │
│    ├────────────────────────▶ Core Service                                  │
│    │                              │                                         │
│    │  7. Generate Access Link     │                                         │
│    └────────────────────────▶ Core Service                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Студенческий флоу: Изучение материала

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STUDENT FLOW: Learning                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Student                                                                    │
│    │                                                                        │
│    │  1. Join by Link                                                       │
│    ├────────────────────────▶ Core Service                                  │
│    │                              │                                         │
│    │  2. Kolb Test                │                                         │
│    ├────────────────────────▶ Personalization Service                       │
│    │                              │                                         │
│    │  3. Enter Interests          │                                         │
│    ├────────────────────────▶ Personalization Service                       │
│    │                              │                                         │
│    │  4. Get Course Map           │                                         │
│    ├────────────────────────▶ KG Service                                    │
│    │                              │                                         │
│    │  5. Open Section             │                                         │
│    ├────────────────────────▶ Personalization Service                       │
│    │                              │                                         │
│    │                              │  5.1 Get Content Essence                │
│    │                              ├────────────────────▶ Content Service    │
│    │                              │                                         │
│    │                              │  5.2 Generate Personalized              │
│    │                              ├────────────────────▶ AI Gateway ──▶ LLM │
│    │                              │                                         │
│    │  6. Ask Companion            │                                         │
│    ├────────────────────────▶ AI Companion Service                          │
│    │                              │                                         │
│    │                              │  6.1 RAG Search                         │
│    │                              ├────────────────────▶ Qdrant             │
│    │                              │                                         │
│    │                              │  6.2 Graph Context                      │
│    │                              ├────────────────────▶ PostgreSQL (AGE)   │
│    │                              │                                         │
│    │                              │  6.3 Generate Response (streaming)      │
│    │                              ├────────────────────▶ AI Gateway ──▶ LLM │
│    │                              │                                         │
│    │  7. Complete Diagnostic      │                                         │
│    ├────────────────────────▶ Assessment Service                            │
│    │                              │                                         │
│    │  8. Update Progress          │                                         │
│    │  ◀────────────────────────── │ ─────────▶ Kafka (event)                │
│    │                              │                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 RAG Pipeline — детальная схема

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           RAG PIPELINE DETAIL                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Query: "Что такое полиморфизм и зачем он нужен?"                      │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        1. QUERY ANALYSIS                            │    │
│  │  • Определение типа вопроса (факт / объяснение / связь)             │    │
│  │  • Извлечение ключевых концептов: ["полиморфизм"]                   │    │
│  │  • Определение intent: объяснение + применение                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        2. HYBRID RETRIEVAL                          │    │
│  │                                                                     │    │
│  │  ┌───────────────────┐         ┌───────────────────┐                │    │
│  │  │   Vector Search   │         │   Graph Lookup    │                │    │
│  │  │   (Qdrant)        │         │   (Apache AGE)    │                │    │
│  │  │                   │         │                   │                │    │
│  │  │  Query embedding  │         │  MATCH (c:Concept │                │    │
│  │  │  → Top-10 chunks  │         │  {title: ~'полим' │                │    │
│  │  │  score > 0.7      │         │  })-[:PREREQ]→(p) │                │    │
│  │  └───────────────────┘         └───────────────────┘                │    │
│  │           │                              │                          │    │
│  │           └──────────────┬───────────────┘                          │    │
│  │                          ▼                                          │    │
│  │  Retrieved:                                                         │    │
│  │  • Chunk 1: "Полиморфизм позволяет..." (score: 0.92)                │    │
│  │  • Chunk 2: "Виды полиморфизма..." (score: 0.88)                    │    │
│  │  • Chunk 3: "Пример использования..." (score: 0.85)                 │    │
│  │  • Graph: Prerequisites = [Наследование, Интерфейсы]                │    │
│  │  • Graph: Related = [Абстракция, Инкапсуляция]                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        3. CONTEXT ASSEMBLY                          │    │
│  │                                                                     │    │
│  │  System: Ты AI-тьютор курса "ООП на Java"                           │    │
│  │                                                                     │    │
│  │  Student Profile:                                                   │    │
│  │  - Стиль Колба: Converger (практика)                                │    │
│  │  - Интересы: [игры, веб-разработка]                                 │    │
│  │  - Уровень: intermediate                                            │    │
│  │                                                                     │    │
│  │  Relevant Content:                                                  │    │
│  │  [Chunk 1] [Chunk 2] [Chunk 3]                                      │    │
│  │                                                                     │    │
│  │  Knowledge Graph Context:                                           │    │
│  │  - Для понимания нужно знать: Наследование ✓, Интерфейсы ✓          │    │
│  │  - Связанные темы: Абстракция, Инкапсуляция                         │    │
│  │                                                                     │    │
│  │  Instructions:                                                      │    │
│  │  - Дай практический пример (Converger)                              │    │
│  │  - Используй аналогию из игр/веб                                    │    │
│  │  - Укажи источник из материалов                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        4. LLM GENERATION                            │    │
│  │                                                                     │    │
│  │  YandexGPT Pro → Streaming Response (SSE)                           │    │
│  │                                                                     │    │
│  │  Response:                                                          │    │
│  │  "Полиморфизм — это способность объектов с одинаковым               │    │
│  │   интерфейсом вести себя по-разному.                                │    │
│  │                                                                     │    │
│  │   Представь систему врагов в игре: у тебя есть базовый              │    │
│  │   класс Enemy с методом attack(). Зомби кусает, скелет              │    │
│  │   стреляет из лука — разное поведение, один интерфейс.              │    │
│  │                                                                     │    │
│  │   В веб-разработке: разные платёжные системы (PayPal,               │    │
│  │   Stripe) реализуют один интерфейс PaymentProcessor..."             │    │
│  │                                                                     │    │
│  │  Source: Раздел 3.2 "Полиморфизм в ООП"                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 Event-Driven: Отслеживание прогресса

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN PROGRESS TRACKING                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Student Actions                    Kafka Topics                            │
│       │                                  │                                  │
│       │  section.opened                  │                                  │
│       ├─────────────────────────────────▶│                                  │
│       │                                  │                                  │
│       │  diagnostic.submitted            │                                  │
│       ├─────────────────────────────────▶│                                  │
│       │                                  │                                  │
│       │  concept.completed               │                                  │
│       ├─────────────────────────────────▶│                                  │
│       │                                  │                                  │
│       │  companion.question.asked        │                                  │
│       ├─────────────────────────────────▶│                                  │
│       │                                  │                                  │
│                                          │                                  │
│                            ┌─────────────┴─────────────┐                    │
│                            ▼                           ▼                    │
│                    ┌──────────────┐           ┌──────────────┐              │
│                    │   Progress   │           │  Analytics   │              │
│                    │   Consumer   │           │   Consumer   │              │
│                    └──────────────┘           └──────────────┘              │
│                            │                           │                    │
│                            ▼                           ▼                    │
│                    ┌──────────────┐           ┌──────────────┐              │
│                    │  PostgreSQL  │           │  ClickHouse  │              │
│                    │  (progress)  │           │  (analytics) │              │
│                    └──────────────┘           └──────────────┘              │
│                                                                             │
│  Event Schema:                                                              │
│  {                                                                          │
│    "event_type": "diagnostic.submitted",                                    │
│    "timestamp": "2024-12-15T10:30:00Z",                                     │
│    "user_id": "uuid",                                                       │
│    "module_id": "uuid",                                                     │
│    "payload": {                                                             │
│      "diagnostic_id": "uuid",                                               │
│      "score": 0.8,                                                          │
│      "time_spent_seconds": 120                                              │
│    }                                                                        │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Деплоймент и инфраструктура

### 6.1 Kubernetes архитектура

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      YANDEX MANAGED KUBERNETES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         NAMESPACE: aico-prod                        │    │
│  │                                                                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │    │
│  │  │auth-service │  │core-service │  │content-svc  │  │  kg-service │ │    │
│  │  │ replicas: 2 │  │ replicas: 3 │  │ replicas: 2 │  │ replicas: 2 │ │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │    │
│  │                                                                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │    │
│  │  │ ai-gateway  │  │ai-companion │  │personal-svc │  │assess-svc   │ │    │
│  │  │ replicas: 3 │  │ replicas: 3 │  │ replicas: 2 │  │ replicas: 2 │ │    │
│  │  │    (HPA)    │  │    (HPA)    │  │             │  │             │ │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │    │
│  │                                                                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │    │
│  │  │analytics-svc│  │notif-service│  │        qdrant               │  │    │
│  │  │ replicas: 2 │  │ replicas: 2 │  │      replicas: 3            │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  External Services (Yandex Cloud Managed):                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  PostgreSQL  │  │    Redis     │  │Data Streams  │  │Object Storage│     │
│  │     (HA)     │  │     (HA)     │  │   (Kafka)    │  │    + CDN     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Конфигурация сервисов

| Сервис | Replicas | CPU Request | Memory Request | HPA |
|--------|----------|-------------|----------------|-----|
| auth-service | 2 | 250m | 512Mi | No |
| core-service | 3 | 500m | 1Gi | No |
| content-service | 2 | 500m | 1Gi | No |
| kg-service | 2 | 500m | 1Gi | No |
| ai-gateway | 2-5 | 500m | 1Gi | Yes (CPU 70%) |
| ai-companion | 2-5 | 1000m | 2Gi | Yes (CPU 70%) |
| personalization-service | 2 | 500m | 1Gi | No |
| assessment-service | 2 | 500m | 1Gi | No |
| analytics-service | 2 | 250m | 512Mi | No |
| notification-service | 2 | 250m | 512Mi | No |
| qdrant | 3 | 1000m | 4Gi | No |

---

## 7. API Contracts (OpenAPI Summary)

### 7.1 Auth Service
```yaml
/auth:
  /register:     POST  # Регистрация пользователя
  /login:        POST  # Аутентификация, получение JWT
  /refresh:      POST  # Обновление access token
  /me:           GET   # Текущий пользователь

/users/{id}:
  /profile:      PUT   # Обновление профиля
```

### 7.2 Core Service
```yaml
/modules:
  /:             POST, GET     # Создание, список модулей
  /{id}:         GET, PUT, DELETE
  /{id}/publish: POST          # Публикация
  /{id}/access-link: POST      # Генерация ссылки
  /{id}/competencies: POST, GET
  /{id}/outcomes: POST, GET
  /{id}/students: GET          # Студенты модуля

/groups:
  /:             POST, GET
  /{id}/join:    POST          # Присоединение по ссылке
```

### 7.3 Content Service
```yaml
/modules/{id}/files:
  /:             POST, GET     # Загрузка, список файлов
  /url:          POST          # Загрузка по URL

/modules/{id}/syllabus: POST   # Загрузка РПД

/sections/{id}:
  /source:       GET           # Исходные чанки
  /additional:   POST          # Доп. контент
```

### 7.4 Knowledge Graph Service
```yaml
/modules/{id}/graph:
  /generate:     POST          # Запуск генерации
  /status:       GET           # Статус генерации
  /:             GET           # Получение графа
  /concepts:     PUT           # Редактирование концептов
  /relations:    PUT           # Редактирование связей
  /confirm:      POST          # Подтверждение
  /route:        GET           # Рекомендованный маршрут

/graph/edit-prompt: POST       # Редактирование через LLM
```

### 7.5 AI Gateway
```yaml
/llm:
  /generate:     POST          # Генерация ответа
  /stream:       POST          # Streaming (SSE)
  /embed:        POST          # Embeddings
  /models:       GET           # Доступные модели
```

### 7.6 AI Companion Service
```yaml
/chat:
  /message:      POST          # Отправка сообщения
  /stream:       WebSocket     # Streaming диалог
  /context:      POST          # Установка контекста страницы
  /history/{module_id}: GET    # История

/content:
  /simplify:     POST          # "Объясни проще"
  /example:      POST          # "Приведи пример"
```

### 7.7 Personalization Service
```yaml
/kolb:
  /questions:    GET           # Вопросы теста
  /submit:       POST          # Отправка ответов

/profile:
  /interests:    PUT           # Обновление интересов
  /companion-style: PUT        # Выбор стиля
  /complexity:   PUT           # Уровень сложности

/modules/{id}/personalize: POST
/sections/{id}/personalized: GET
```

### 7.8 Assessment Service
```yaml
/sections/{id}/diagnostics: POST, GET
/diagnostics:
  /generate:     POST          # Автогенерация
  /{id}:         GET
  /{id}/submit:  POST          # Отправка ответов

/concepts/{id}/case: POST
/case:
  /generate:     POST
  /{id}/start:   POST
  /{id}/answer:  POST
  /{id}/result:  GET

/students/{id}/results: GET
/modules/{id}/results: GET
```

### 7.9 Analytics Service
```yaml
/modules/{id}:
  /status-matrix: GET
  /students/{sid}/details: GET
  /students/{sid}/questions: GET
  /analytics:    GET

/students/{id}/portfolio: GET
```

---

## 8. Резюме архитектурных решений

| Решение | Выбор | Обоснование |
|---------|-------|-------------|
| **Архитектура** | Микросервисы | Независимое масштабирование AI-компонентов, изоляция отказов, параллельная разработка |
| **API Gateway** | Yandex API Gateway | Serverless, нативная интеграция, WebSocket support |
| **Primary LLM** | YandexGPT Pro | Интеграция с Yandex Cloud, русский язык, совместимый API |
| **Vector DB** | Qdrant (self-hosted) | Лучшая latency, metadata filtering, hybrid search |
| **Graph DB** | PostgreSQL + Apache AGE | Единая инфраструктура, Cypher queries, достаточно для масштаба |
| **Event Bus** | Yandex Data Streams | Managed Kafka API для event sourcing прогресса |
| **Backend** | FastAPI (Python) | Async, ML-экосистема, автодокументация |
| **Frontend** | React + TypeScript | Экосистема, библиотеки для графов и редакторов |
| **Streaming** | SSE | Проще WebSocket для LLM streaming, автоматический reconnect |