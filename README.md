# Техническая архитектура и алгоритм универсальной миграции баз данных (SQL ↔ NoSQL)

Разработанная система представляет собой двунаправленный ETL-конвейер (Extract, Transform, Load), решающий фундаментальную проблему объектно-реляционного несоответствия (Object-Relational Impedance Mismatch). В основе системы лежит схемо-независимый (schema-agnostic) подход. Он опирается на топологический анализ графов, чтение системных каталогов СУБД и эвристическое профилирование данных.

Ниже представлен детальный разбор работы алгоритма, описывающий каждый шаг принятия решений системой.

---

## Глава 1. Трансформация SQL в MongoDB: Алгоритм инкапсуляции

Процесс перевода Третьей Нормальной Формы (3НФ) в глубоко вложенные агрегаты NoSQL базируется на концепциях предметно-ориентированного проектирования (Domain-Driven Design). Скрипт должен алгоритмически определить, какая таблица становится корневой коллекцией, а какая — вложенным документом.

### 1.1 Обнаружение связующих таблиц (Many-to-Many)

На первом шаге скрипт опрашивает базу данных через инспектор `sqlalchemy.inspect`, извлекая списки таблиц, внешних ключей (Foreign Keys) и первичных ключей (Primary Keys). Алгоритм сканирует каждую таблицу, проверяя следующее условие: если таблица содержит два или более внешних ключа, и все эти внешние ключи входят в состав её составного первичного ключа (Composite PK), то система помечает данную таблицу как `Junction Table` (связующую). Такие таблицы исключаются из дальнейшего поиска "родителей", так как их данные будут преобразованы в двунаправленные массивы внутри связываемых сущностей.

### 1.2 Построение графа зависимостей и расчет весов

Для оставшихся таблиц алгоритм строит ориентированный граф. Для каждой вершины (таблицы) вычисляются два математических параметра:

- **In-Degree (Входящая степень):** Количество внешних ключей из других таблиц, ссылающихся на данную.
- **Out-Degree (Исходящая степень):** Количество внешних ключей внутри самой таблицы, ссылающихся на другие сущности.

Опираясь на эти метрики, алгоритм понимает базовую топологию. Например, если у таблицы `Genre` (Жанры) In-Degree > 0, но Out-Degree = 0, алгоритм идентифицирует её как статичный "Справочник" (Dictionary).

### 1.3 Эвристика выбора Истинного Родителя (Aggregate Root Resolution)

Это ядро алгоритма SQL to Mongo. У одной таблицы может быть несколько связей (например, `Track` ссылается на `Album` и `Genre`). Чтобы решить, куда встроить `Track`, алгоритм формирует список кандидатов и вычисляет вес каждого кандидата по четырем критериям:

1. **Идентифицирующая связь (`is_identifying`):** Скрипт проверяет, входит ли внешний ключ кандидата в состав первичного ключа дочерней таблицы. Если да, это жесткая связь "сущность-зависимость" (наивысший приоритет).
2. **Композиция (`is_composition`):** Скрипт анализирует имена. Если имя колонки внешнего ключа совпадает с паттерном `<имя_родительской_таблицы>_id` (например, `artist_id` указывает на таблицу `Artist`), алгоритм маркирует это как логическую композицию "Часть-Целое".
3. **Справочник (`is_dictionary`):** Используя вычисленный ранее Out-Degree = 0, алгоритм пессимизирует справочники. Система понимает, что встраивать миллион треков внутрь справочника из 10 жанров — это нарушение паттернов NoSQL.
4. **Вес (`weight`):** Равен In-Degree родителя. При прочих равных выбирается родитель с меньшим числом связей, чтобы не допустить раздувания BSON-документа сверх лимита в 16 МБ.

Кандидаты сортируются по этим критериям. Таблицы, которые не были встроены ни в одного кандидата, объявляются Истинными Корнями (Root Entities) и становятся отдельными коллекциями в MongoDB.

### 1.4 Рекурсивная сборка дерева и защита от циклов

Для переноса данных алгоритм читает корневые таблицы пакетами (chunks). Для каждого пакета извлекается массив ID. Затем вызывается рекурсивная функция `build_nested_tree`.
Чтобы избежать проблемы "N+1 запросов", функция формирует единый SQL-запрос для всех потомков: `SELECT * FROM child WHERE parent_id IN (...)`. Результаты группируются в памяти.
Чтобы скрипт не попал в бесконечный цикл (когда таблица ссылается сама на себя, например, рекурсия сотрудников), в функцию передается множество `visited_tables`. Перед обработкой дочерней таблицы скрипт проверяет её наличие в множестве. Если она там есть, ветвь обрывается.

---

## Глава 2. Трансформация MongoDB в SQL: Алгоритм расщепления

Обратная миграция из schemaless (бессхемной) структуры в строгую реляционную модель требует парсинга на лету и предотвращения избыточной нормализации.

### 2.1 Рекурсивный парсинг документов на основе типов

Скрипт поочередно читает документы MongoDB и прогоняет их через функцию `_parse_mongo_doc`. Функция анализирует тип (typeof) каждого ключа и принимает архитектурные решения:

- **Атомарные типы (String, Number):** Очищаются и добавляются как колонки в текущую строку.
- **Вложенный объект (Dict):** Алгоритм распознает связь "один-к-одному" (1:1). Согласно правилам 3НФ, такие данные (например, `imdb: {rating: 5}`) не должны выделяться в отдельную таблицу, если они зависят только от первичного ключа фильма. Поэтому скрипт "сплющивает" (flatten) объект, рекурсивно поднимая его поля в родительскую строку с префиксом `imdb_rating`.
- **Массив объектов (List[Dict]):** Распознается как связь "один-ко-многим" (1:N). Алгоритм создает новую дочернюю таблицу, прокидывая в каждую строку сгенерированный `parent_uuid` для сохранения ссылки на родителя.
- **Массив примитивов (List[String/Int]):** Распознается как денормализованная связь "многие-ко-многим" (M:N). Алгоритм генерирует связующую (Junction) таблицу, состоящую из `parent_uuid` и колонки `target_val`.

### 2.2 Эволюция DDL-схемы (Schema Evolution)

Так как схема в MongoDB может мутировать, скрипт поддерживает актуальную структуру таблиц в оперативной памяти. При обнаружении нового ключа в документе, функция `_infer_sql_type` анализирует значение (приводя целые числа к BIGINT, числа с плавающей точкой к DOUBLE PRECISION). Система приостанавливает накопление пакета данных и выполняет инъекцию DDL-команды `ALTER TABLE ... ADD COLUMN ...` в PostgreSQL.

### 2.3 Фильтрация аномалий (Buffer Storm)

Особое внимание уделено защите от загрязнения схемы. Драйверы MongoDB часто экспортируют `ObjectId` в виде сырых бинарных массивов (Buffer). Если классический парсер попытается рекурсивно обойти такой объект, он создаст десятки мусорных SQL-колонок (например, `_id_buffer_readUInt8`). Алгоритм блокирует это на стадии предварительной очистки, жестко перехватывая объекты с типом `ObjectId` и сериализуя их в строковый формат.

---

## Глава 3. Восстановление связей и Профилирование данных

После миграции MongoDB в SQL, система имеет набор физически изолированных таблиц. Модуль `DataAnalyzer.enforce_physical_fks` берет на себя задачу автоматического восстановления `FOREIGN KEY`.

### 3.1 Структурное связывание

В первую очередь скрипт ищет колонку `parent_uuid`, которую сам сгенерировал при расщеплении вложенных объектов. Понимая номенклатуру (имя дочерней таблицы включает имя родителя), скрипт находит родительскую таблицу и устанавливает физическую связь.

### 3.2 Логическое связывание (Data Profiling)

Самый сложный этап — восстановление независимых связей, которые в NoSQL хранились просто как строковые значения (например, `account_id` внутри коллекции `orders`).
Скрипт сканирует все колонки, оканчивающиеся на `_id`. Чтобы убедиться в наличии связи, алгоритм делает выборку (sample) 100 уникальных непустых значений из дочерней колонки. Затем он ищет родительские таблицы, в которых есть колонка с похожим именем или колонка `_id`. Алгоритм отправляет тестовый SQL-запрос `WHERE id IN (sample_values)`. Если находится пересечение данных, скрипт принимает решение о создании `FOREIGN KEY`.

**Эвристика бизнес-ключей:** Алгоритм способен обходить плохой дизайн БД. Если он анализирует колонку `user_id`, но видит в семпле символ `@` (паттерн электронной почты), он динамически добавляет колонку `email` в родительских таблицах в список потенциальных целей для связи.

---

## Глава 4. Оценка сложности алгоритма

- **Временная сложность (Топология):** Вычисление весов графа и поиск связей выполняется за $O(V + E)$, где $V$ — количество таблиц, а $E$ — количество связей.
- **Временная сложность (Чтение/Запись):** Потоковая обработка ограничена $O(B \cdot D \cdot T)$, где $B$ — размер чанка, $D$ — глубина рекурсии, а $T$ — время выполнения IN-запроса. Использование `yield_per` делает эту сложность линейной по отношению к объему БД.
- **Пространственная сложность (RAM):** Жестко зафиксирована на уровне $O(B \cdot \text{Max Doc Size})$. Независимо от размера базы данных (10 ГБ или 100 ТБ), скрипт потребляет константный объем памяти, что делает его Enterprise-ready.
- **Сложность профилирования:** Восстановление связей имеет сложность $O(V^2 \cdot \log R)$. Оптимизация (LIMIT 100 для семплирования) позволяет выполнять эту операцию за приемлемое время без сканирования таблиц целиком (Full Table Scan).

## Рез-ты работы MONGO TO SQL
<img width="1358" height="679" alt="1" src="https://github.com/user-attachments/assets/6fbbae95-c830-4446-96b6-22eafe1d1e54" />

MONGO:
{
  comments: {
    _id: 'object',
    '_id.buffer': 'object',
    '_id.buffer.0': 'number',
    '_id.buffer.1': 'number',
    '_id.buffer.2': 'number',
    '_id.buffer.3': 'number',
    '_id.buffer.4': 'number',
    '_id.buffer.5': 'number',
    '_id.buffer.6': 'number',
    '_id.buffer.7': 'number',
    '_id.buffer.8': 'number',
    '_id.buffer.9': 'number',
    '_id.buffer.10': 'number',
    '_id.buffer.11': 'number',
    '_id.buffer.readBigUInt64LE': 'function',
    '_id.buffer.readBigUInt64BE': 'function',
    '_id.buffer.readBigUint64LE': 'function',
    '_id.buffer.readBigUint64BE': 'function',
    '_id.buffer.readBigInt64LE': 'function',
    '_id.buffer.readBigInt64BE': 'function',
    '_id.buffer.writeBigUInt64LE': 'function',
    '_id.buffer.writeBigUInt64BE': 'function',
    '_id.buffer.writeBigUint64LE': 'function',
    '_id.buffer.writeBigUint64BE': 'function',
    '_id.buffer.writeBigInt64LE': 'function',
    '_id.buffer.writeBigInt64BE': 'function',
    '_id.buffer.readUIntLE': 'function',
    '_id.buffer.readUInt32LE': 'function',
    '_id.buffer.readUInt16LE': 'function',
    '_id.buffer.readUInt8': 'function',
    '_id.buffer.readUIntBE': 'function',
    '_id.buffer.readUInt32BE': 'function',
    '_id.buffer.readUInt16BE': 'function',
    '_id.buffer.readUintLE': 'function',
    '_id.buffer.readUint32LE': 'function',
    '_id.buffer.readUint16LE': 'function',
    '_id.buffer.readUint8': 'function',
    '_id.buffer.readUintBE': 'function',
    '_id.buffer.readUint32BE': 'function',
    '_id.buffer.readUint16BE': 'function',
    '_id.buffer.readIntLE': 'function',
    '_id.buffer.readInt32LE': 'function',
    '_id.buffer.readInt16LE': 'function',
    '_id.buffer.readInt8': 'function',
    '_id.buffer.readIntBE': 'function',
    '_id.buffer.readInt32BE': 'function',
    '_id.buffer.readInt16BE': 'function',
    '_id.buffer.writeUIntLE': 'function',
    '_id.buffer.writeUInt32LE': 'function',
    '_id.buffer.writeUInt16LE': 'function',
    '_id.buffer.writeUInt8': 'function',
    '_id.buffer.writeUIntBE': 'function',
    '_id.buffer.writeUInt32BE': 'function',
    '_id.buffer.writeUInt16BE': 'function',
    '_id.buffer.writeUintLE': 'function',
    '_id.buffer.writeUint32LE': 'function',
    '_id.buffer.writeUint16LE': 'function',
    '_id.buffer.writeUint8': 'function',
    '_id.buffer.writeUintBE': 'function',
    '_id.buffer.writeUint32BE': 'function',
    '_id.buffer.writeUint16BE': 'function',
    '_id.buffer.writeIntLE': 'function',
    '_id.buffer.writeInt32LE': 'function',
    '_id.buffer.writeInt16LE': 'function',
    '_id.buffer.writeInt8': 'function',
    '_id.buffer.writeIntBE': 'function',
    '_id.buffer.writeInt32BE': 'function',
    '_id.buffer.writeInt16BE': 'function',
    '_id.buffer.readFloatLE': 'function',
    '_id.buffer.readFloatBE': 'function',
    '_id.buffer.readDoubleLE': 'function',
    '_id.buffer.readDoubleBE': 'function',
    '_id.buffer.writeFloatLE': 'function',
    '_id.buffer.writeFloatBE': 'function',
    '_id.buffer.writeDoubleLE': 'function',
    '_id.buffer.writeDoubleBE': 'function',
    '_id.buffer.asciiSlice': 'function',
    '_id.buffer.base64Slice': 'function',
    '_id.buffer.base64urlSlice': 'function',
    '_id.buffer.latin1Slice': 'function',
    '_id.buffer.hexSlice': 'function',
    '_id.buffer.ucs2Slice': 'function',
    '_id.buffer.utf8Slice': 'function',
    '_id.buffer.asciiWrite': 'function',
    '_id.buffer.base64Write': 'function',
    '_id.buffer.base64urlWrite': 'function',
    '_id.buffer.latin1Write': 'function',
    '_id.buffer.hexWrite': 'function',
    '_id.buffer.ucs2Write': 'function',
    '_id.buffer.utf8Write': 'function',
    '_id.buffer.parent': 'object',
    '_id.buffer.offset': 'number',
    '_id.buffer.copy': 'function',
    '_id.buffer.toString': 'function',
    '_id.buffer.equals': 'function',
    '_id.buffer.inspect': 'function',
    '_id.buffer.compare': 'function',
    '_id.buffer.indexOf': 'function',
    '_id.buffer.lastIndexOf': 'function',
    '_id.buffer.includes': 'function',
    '_id.buffer.fill': 'function',
    '_id.buffer.write': 'function',
    '_id.buffer.toJSON': 'function',
    '_id.buffer.subarray': 'function',
    '_id.buffer.slice': 'function',
    '_id.buffer.swap16': 'function',
    '_id.buffer.swap32': 'function',
    '_id.buffer.swap64': 'function',
    '_id.buffer.toLocaleString': 'function',
    '_id.serverVersions': 'array',
    '_id.platforms': 'array',
    '_id.topologies': 'array',
    '_id.help': 'function',
    name: 'string',
    email: 'string',
    movie_id: 'object',
    'movie_id.buffer': 'object',
    'movie_id.buffer.0': 'number',
    'movie_id.buffer.1': 'number',
    'movie_id.buffer.2': 'number',
    'movie_id.buffer.3': 'number',
    'movie_id.buffer.4': 'number',
    'movie_id.buffer.5': 'number',
    'movie_id.buffer.6': 'number',
    'movie_id.buffer.7': 'number',
    'movie_id.buffer.8': 'number',
    'movie_id.buffer.9': 'number',
    'movie_id.buffer.10': 'number',
    'movie_id.buffer.11': 'number',
    'movie_id.buffer.readBigUInt64LE': 'function',
    'movie_id.buffer.readBigUInt64BE': 'function',
    'movie_id.buffer.readBigUint64LE': 'function',
    'movie_id.buffer.readBigUint64BE': 'function',
    'movie_id.buffer.readBigInt64LE': 'function',
    'movie_id.buffer.readBigInt64BE': 'function',
    'movie_id.buffer.writeBigUInt64LE': 'function',
    'movie_id.buffer.writeBigUInt64BE': 'function',
    'movie_id.buffer.writeBigUint64LE': 'function',
    'movie_id.buffer.writeBigUint64BE': 'function',
    'movie_id.buffer.writeBigInt64LE': 'function',
    'movie_id.buffer.writeBigInt64BE': 'function',
    'movie_id.buffer.readUIntLE': 'function',
    'movie_id.buffer.readUInt32LE': 'function',
    'movie_id.buffer.readUInt16LE': 'function',
    'movie_id.buffer.readUInt8': 'function',
    'movie_id.buffer.readUIntBE': 'function',
    'movie_id.buffer.readUInt32BE': 'function',
    'movie_id.buffer.readUInt16BE': 'function',
    'movie_id.buffer.readUintLE': 'function',
    'movie_id.buffer.readUint32LE': 'function',
    'movie_id.buffer.readUint16LE': 'function',
    'movie_id.buffer.readUint8': 'function',
    'movie_id.buffer.readUintBE': 'function',
    'movie_id.buffer.readUint32BE': 'function',
    'movie_id.buffer.readUint16BE': 'function',
    'movie_id.buffer.readIntLE': 'function',
    'movie_id.buffer.readInt32LE': 'function',
    'movie_id.buffer.readInt16LE': 'function',
    'movie_id.buffer.readInt8': 'function',
    'movie_id.buffer.readIntBE': 'function',
    'movie_id.buffer.readInt32BE': 'function',
    'movie_id.buffer.readInt16BE': 'function',
    'movie_id.buffer.writeUIntLE': 'function',
    'movie_id.buffer.writeUInt32LE': 'function',
    'movie_id.buffer.writeUInt16LE': 'function',
    'movie_id.buffer.writeUInt8': 'function',
    'movie_id.buffer.writeUIntBE': 'function',
    'movie_id.buffer.writeUInt32BE': 'function',
    'movie_id.buffer.writeUInt16BE': 'function',
    'movie_id.buffer.writeUintLE': 'function',
    'movie_id.buffer.writeUint32LE': 'function',
    'movie_id.buffer.writeUint16LE': 'function',
    'movie_id.buffer.writeUint8': 'function',
    'movie_id.buffer.writeUintBE': 'function',
    'movie_id.buffer.writeUint32BE': 'function',
    'movie_id.buffer.writeUint16BE': 'function',
    'movie_id.buffer.writeIntLE': 'function',
    'movie_id.buffer.writeInt32LE': 'function',
    'movie_id.buffer.writeInt16LE': 'function',
    'movie_id.buffer.writeInt8': 'function',
    'movie_id.buffer.writeIntBE': 'function',
    'movie_id.buffer.writeInt32BE': 'function',
    'movie_id.buffer.writeInt16BE': 'function',
    'movie_id.buffer.readFloatLE': 'function',
    'movie_id.buffer.readFloatBE': 'function',
    'movie_id.buffer.readDoubleLE': 'function',
    'movie_id.buffer.readDoubleBE': 'function',
    'movie_id.buffer.writeFloatLE': 'function',
    'movie_id.buffer.writeFloatBE': 'function',
    'movie_id.buffer.writeDoubleLE': 'function',
    'movie_id.buffer.writeDoubleBE': 'function',
    'movie_id.buffer.asciiSlice': 'function',
    'movie_id.buffer.base64Slice': 'function',
    'movie_id.buffer.base64urlSlice': 'function',
    'movie_id.buffer.latin1Slice': 'function',
    'movie_id.buffer.hexSlice': 'function',
    'movie_id.buffer.ucs2Slice': 'function',
    'movie_id.buffer.utf8Slice': 'function',
    'movie_id.buffer.asciiWrite': 'function',
    'movie_id.buffer.base64Write': 'function',
    'movie_id.buffer.base64urlWrite': 'function',
    'movie_id.buffer.latin1Write': 'function',
    'movie_id.buffer.hexWrite': 'function',
    'movie_id.buffer.ucs2Write': 'function',
    'movie_id.buffer.utf8Write': 'function',
    'movie_id.buffer.parent': 'object',
    'movie_id.buffer.offset': 'number',
    'movie_id.buffer.copy': 'function',
    'movie_id.buffer.toString': 'function',
    'movie_id.buffer.equals': 'function',
    'movie_id.buffer.inspect': 'function',
    'movie_id.buffer.compare': 'function',
    'movie_id.buffer.indexOf': 'function',
    'movie_id.buffer.lastIndexOf': 'function',
    'movie_id.buffer.includes': 'function',
    'movie_id.buffer.fill': 'function',
    'movie_id.buffer.write': 'function',
    'movie_id.buffer.toJSON': 'function',
    'movie_id.buffer.subarray': 'function',
    'movie_id.buffer.slice': 'function',
    'movie_id.buffer.swap16': 'function',
    'movie_id.buffer.swap32': 'function',
    'movie_id.buffer.swap64': 'function',
    'movie_id.buffer.toLocaleString': 'function',
    'movie_id.serverVersions': 'array',
    'movie_id.platforms': 'array',
    'movie_id.topologies': 'array',
    'movie_id.help': 'function',
    text: 'string',
    date: 'object'
  },
  movies: {
    _id: 'object',
    '_id.buffer': 'object',
    '_id.buffer.0': 'number',
    '_id.buffer.1': 'number',
    '_id.buffer.2': 'number',
    '_id.buffer.3': 'number',
    '_id.buffer.4': 'number',
    '_id.buffer.5': 'number',
    '_id.buffer.6': 'number',
    '_id.buffer.7': 'number',
    '_id.buffer.8': 'number',
    '_id.buffer.9': 'number',
    '_id.buffer.10': 'number',
    '_id.buffer.11': 'number',
    '_id.buffer.readBigUInt64LE': 'function',
    '_id.buffer.readBigUInt64BE': 'function',
    '_id.buffer.readBigUint64LE': 'function',
    '_id.buffer.readBigUint64BE': 'function',
    '_id.buffer.readBigInt64LE': 'function',
    '_id.buffer.readBigInt64BE': 'function',
    '_id.buffer.writeBigUInt64LE': 'function',
    '_id.buffer.writeBigUInt64BE': 'function',
    '_id.buffer.writeBigUint64LE': 'function',
    '_id.buffer.writeBigUint64BE': 'function',
    '_id.buffer.writeBigInt64LE': 'function',
    '_id.buffer.writeBigInt64BE': 'function',
    '_id.buffer.readUIntLE': 'function',
    '_id.buffer.readUInt32LE': 'function',
    '_id.buffer.readUInt16LE': 'function',
    '_id.buffer.readUInt8': 'function',
    '_id.buffer.readUIntBE': 'function',
    '_id.buffer.readUInt32BE': 'function',
    '_id.buffer.readUInt16BE': 'function',
    '_id.buffer.readUintLE': 'function',
    '_id.buffer.readUint32LE': 'function',
    '_id.buffer.readUint16LE': 'function',
    '_id.buffer.readUint8': 'function',
    '_id.buffer.readUintBE': 'function',
    '_id.buffer.readUint32BE': 'function',
    '_id.buffer.readUint16BE': 'function',
    '_id.buffer.readIntLE': 'function',
    '_id.buffer.readInt32LE': 'function',
    '_id.buffer.readInt16LE': 'function',
    '_id.buffer.readInt8': 'function',
    '_id.buffer.readIntBE': 'function',
    '_id.buffer.readInt32BE': 'function',
    '_id.buffer.readInt16BE': 'function',
    '_id.buffer.writeUIntLE': 'function',
    '_id.buffer.writeUInt32LE': 'function',
    '_id.buffer.writeUInt16LE': 'function',
    '_id.buffer.writeUInt8': 'function',
    '_id.buffer.writeUIntBE': 'function',
    '_id.buffer.writeUInt32BE': 'function',
    '_id.buffer.writeUInt16BE': 'function',
    '_id.buffer.writeUintLE': 'function',
    '_id.buffer.writeUint32LE': 'function',
    '_id.buffer.writeUint16LE': 'function',
    '_id.buffer.writeUint8': 'function',
    '_id.buffer.writeUintBE': 'function',
    '_id.buffer.writeUint32BE': 'function',
    '_id.buffer.writeUint16BE': 'function',
    '_id.buffer.writeIntLE': 'function',
    '_id.buffer.writeInt32LE': 'function',
    '_id.buffer.writeInt16LE': 'function',
    '_id.buffer.writeInt8': 'function',
    '_id.buffer.writeIntBE': 'function',
    '_id.buffer.writeInt32BE': 'function',
    '_id.buffer.writeInt16BE': 'function',
    '_id.buffer.readFloatLE': 'function',
    '_id.buffer.readFloatBE': 'function',
    '_id.buffer.readDoubleLE': 'function',
    '_id.buffer.readDoubleBE': 'function',
    '_id.buffer.writeFloatLE': 'function',
    '_id.buffer.writeFloatBE': 'function',
    '_id.buffer.writeDoubleLE': 'function',
    '_id.buffer.writeDoubleBE': 'function',
    '_id.buffer.asciiSlice': 'function',
    '_id.buffer.base64Slice': 'function',
    '_id.buffer.base64urlSlice': 'function',
    '_id.buffer.latin1Slice': 'function',
    '_id.buffer.hexSlice': 'function',
    '_id.buffer.ucs2Slice': 'function',
    '_id.buffer.utf8Slice': 'function',
    '_id.buffer.asciiWrite': 'function',
    '_id.buffer.base64Write': 'function',
    '_id.buffer.base64urlWrite': 'function',
    '_id.buffer.latin1Write': 'function',
    '_id.buffer.hexWrite': 'function',
    '_id.buffer.ucs2Write': 'function',
    '_id.buffer.utf8Write': 'function',
    '_id.buffer.parent': 'object',
    '_id.buffer.offset': 'number',
    '_id.buffer.copy': 'function',
    '_id.buffer.toString': 'function',
    '_id.buffer.equals': 'function',
    '_id.buffer.inspect': 'function',
    '_id.buffer.compare': 'function',
    '_id.buffer.indexOf': 'function',
    '_id.buffer.lastIndexOf': 'function',
    '_id.buffer.includes': 'function',
    '_id.buffer.fill': 'function',
    '_id.buffer.write': 'function',
    '_id.buffer.toJSON': 'function',
    '_id.buffer.subarray': 'function',
    '_id.buffer.slice': 'function',
    '_id.buffer.swap16': 'function',
    '_id.buffer.swap32': 'function',
    '_id.buffer.swap64': 'function',
    '_id.buffer.toLocaleString': 'function',
    '_id.serverVersions': 'array',
    '_id.platforms': 'array',
    '_id.topologies': 'array',
    '_id.help': 'function',
    plot: 'string',
    genres: 'array',
    runtime: 'number',
    cast: 'array',
    num_mflix_comments: 'number',
    title: 'string',
    fullplot: 'string',
    countries: 'array',
    released: 'object',
    directors: 'array',
    rated: 'string',
    awards: 'object',
    'awards.wins': 'number',
    'awards.nominations': 'number',
    'awards.text': 'string',
    lastupdated: 'string',
    year: 'number',
    imdb: 'object',
    'imdb.rating': 'number',
    'imdb.votes': 'number',
    'imdb.id': 'number',
    type: 'string',
    tomatoes: 'object',
    'tomatoes.viewer': 'object',
    'tomatoes.viewer.rating': 'number',
    'tomatoes.viewer.numReviews': 'number',
    'tomatoes.viewer.meter': 'number',
    'tomatoes.lastUpdated': 'object'
  },
  sessoins: null,
  theaters: {
    _id: 'object',
    '_id.buffer': 'object',
    '_id.buffer.0': 'number',
    '_id.buffer.1': 'number',
    '_id.buffer.2': 'number',
    '_id.buffer.3': 'number',
    '_id.buffer.4': 'number',
    '_id.buffer.5': 'number',
    '_id.buffer.6': 'number',
    '_id.buffer.7': 'number',
    '_id.buffer.8': 'number',
    '_id.buffer.9': 'number',
    '_id.buffer.10': 'number',
    '_id.buffer.11': 'number',
    '_id.buffer.readBigUInt64LE': 'function',
    '_id.buffer.readBigUInt64BE': 'function',
    '_id.buffer.readBigUint64LE': 'function',
    '_id.buffer.readBigUint64BE': 'function',
    '_id.buffer.readBigInt64LE': 'function',
    '_id.buffer.readBigInt64BE': 'function',
    '_id.buffer.writeBigUInt64LE': 'function',
    '_id.buffer.writeBigUInt64BE': 'function',
    '_id.buffer.writeBigUint64LE': 'function',
    '_id.buffer.writeBigUint64BE': 'function',
    '_id.buffer.writeBigInt64LE': 'function',
    '_id.buffer.writeBigInt64BE': 'function',
    '_id.buffer.readUIntLE': 'function',
    '_id.buffer.readUInt32LE': 'function',
    '_id.buffer.readUInt16LE': 'function',
    '_id.buffer.readUInt8': 'function',
    '_id.buffer.readUIntBE': 'function',
    '_id.buffer.readUInt32BE': 'function',
    '_id.buffer.readUInt16BE': 'function',
    '_id.buffer.readUintLE': 'function',
    '_id.buffer.readUint32LE': 'function',
    '_id.buffer.readUint16LE': 'function',
    '_id.buffer.readUint8': 'function',
    '_id.buffer.readUintBE': 'function',
    '_id.buffer.readUint32BE': 'function',
    '_id.buffer.readUint16BE': 'function',
    '_id.buffer.readIntLE': 'function',
    '_id.buffer.readInt32LE': 'function',
    '_id.buffer.readInt16LE': 'function',
    '_id.buffer.readInt8': 'function',
    '_id.buffer.readIntBE': 'function',
    '_id.buffer.readInt32BE': 'function',
    '_id.buffer.readInt16BE': 'function',
    '_id.buffer.writeUIntLE': 'function',
    '_id.buffer.writeUInt32LE': 'function',
    '_id.buffer.writeUInt16LE': 'function',
    '_id.buffer.writeUInt8': 'function',
    '_id.buffer.writeUIntBE': 'function',
    '_id.buffer.writeUInt32BE': 'function',
    '_id.buffer.writeUInt16BE': 'function',
    '_id.buffer.writeUintLE': 'function',
    '_id.buffer.writeUint32LE': 'function',
    '_id.buffer.writeUint16LE': 'function',
    '_id.buffer.writeUint8': 'function',
    '_id.buffer.writeUintBE': 'function',
    '_id.buffer.writeUint32BE': 'function',
    '_id.buffer.writeUint16BE': 'function',
    '_id.buffer.writeIntLE': 'function',
    '_id.buffer.writeInt32LE': 'function',
    '_id.buffer.writeInt16LE': 'function',
    '_id.buffer.writeInt8': 'function',
    '_id.buffer.writeIntBE': 'function',
    '_id.buffer.writeInt32BE': 'function',
    '_id.buffer.writeInt16BE': 'function',
    '_id.buffer.readFloatLE': 'function',
    '_id.buffer.readFloatBE': 'function',
    '_id.buffer.readDoubleLE': 'function',
    '_id.buffer.readDoubleBE': 'function',
    '_id.buffer.writeFloatLE': 'function',
    '_id.buffer.writeFloatBE': 'function',
    '_id.buffer.writeDoubleLE': 'function',
    '_id.buffer.writeDoubleBE': 'function',
    '_id.buffer.asciiSlice': 'function',
    '_id.buffer.base64Slice': 'function',
    '_id.buffer.base64urlSlice': 'function',
    '_id.buffer.latin1Slice': 'function',
    '_id.buffer.hexSlice': 'function',
    '_id.buffer.ucs2Slice': 'function',
    '_id.buffer.utf8Slice': 'function',
    '_id.buffer.asciiWrite': 'function',
    '_id.buffer.base64Write': 'function',
    '_id.buffer.base64urlWrite': 'function',
    '_id.buffer.latin1Write': 'function',
    '_id.buffer.hexWrite': 'function',
    '_id.buffer.ucs2Write': 'function',
    '_id.buffer.utf8Write': 'function',
    '_id.buffer.parent': 'object',
    '_id.buffer.offset': 'number',
    '_id.buffer.copy': 'function',
    '_id.buffer.toString': 'function',
    '_id.buffer.equals': 'function',
    '_id.buffer.inspect': 'function',
    '_id.buffer.compare': 'function',
    '_id.buffer.indexOf': 'function',
    '_id.buffer.lastIndexOf': 'function',
    '_id.buffer.includes': 'function',
    '_id.buffer.fill': 'function',
    '_id.buffer.write': 'function',
    '_id.buffer.toJSON': 'function',
    '_id.buffer.subarray': 'function',
    '_id.buffer.slice': 'function',
    '_id.buffer.swap16': 'function',
    '_id.buffer.swap32': 'function',
    '_id.buffer.swap64': 'function',
    '_id.buffer.toLocaleString': 'function',
    '_id.serverVersions': 'array',
    '_id.platforms': 'array',
    '_id.topologies': 'array',
    '_id.help': 'function',
    theaterId: 'number',
    location: 'object',
    'location.address': 'object',
    'location.address.street1': 'string',
    'location.address.city': 'string',
    'location.address.state': 'string',
    'location.address.zipcode': 'string',
    'location.geo': 'object',
    'location.geo.type': 'string',
    'location.geo.coordinates': 'array'
  },
  users: {
    _id: 'object',
    '_id.buffer': 'object',
    '_id.buffer.0': 'number',
    '_id.buffer.1': 'number',
    '_id.buffer.2': 'number',
    '_id.buffer.3': 'number',
    '_id.buffer.4': 'number',
    '_id.buffer.5': 'number',
    '_id.buffer.6': 'number',
    '_id.buffer.7': 'number',
    '_id.buffer.8': 'number',
    '_id.buffer.9': 'number',
    '_id.buffer.10': 'number',
    '_id.buffer.11': 'number',
    '_id.buffer.readBigUInt64LE': 'function',
    '_id.buffer.readBigUInt64BE': 'function',
    '_id.buffer.readBigUint64LE': 'function',
    '_id.buffer.readBigUint64BE': 'function',
    '_id.buffer.readBigInt64LE': 'function',
    '_id.buffer.readBigInt64BE': 'function',
    '_id.buffer.writeBigUInt64LE': 'function',
    '_id.buffer.writeBigUInt64BE': 'function',
    '_id.buffer.writeBigUint64LE': 'function',
    '_id.buffer.writeBigUint64BE': 'function',
    '_id.buffer.writeBigInt64LE': 'function',
    '_id.buffer.writeBigInt64BE': 'function',
    '_id.buffer.readUIntLE': 'function',
    '_id.buffer.readUInt32LE': 'function',
    '_id.buffer.readUInt16LE': 'function',
    '_id.buffer.readUInt8': 'function',
    '_id.buffer.readUIntBE': 'function',
    '_id.buffer.readUInt32BE': 'function',
    '_id.buffer.readUInt16BE': 'function',
    '_id.buffer.readUintLE': 'function',
    '_id.buffer.readUint32LE': 'function',
    '_id.buffer.readUint16LE': 'function',
    '_id.buffer.readUint8': 'function',
    '_id.buffer.readUintBE': 'function',
    '_id.buffer.readUint32BE': 'function',
    '_id.buffer.readUint16BE': 'function',
    '_id.buffer.readIntLE': 'function',
    '_id.buffer.readInt32LE': 'function',
    '_id.buffer.readInt16LE': 'function',
    '_id.buffer.readInt8': 'function',
    '_id.buffer.readIntBE': 'function',
    '_id.buffer.readInt32BE': 'function',
    '_id.buffer.readInt16BE': 'function',
    '_id.buffer.writeUIntLE': 'function',
    '_id.buffer.writeUInt32LE': 'function',
    '_id.buffer.writeUInt16LE': 'function',
    '_id.buffer.writeUInt8': 'function',
    '_id.buffer.writeUIntBE': 'function',
    '_id.buffer.writeUInt32BE': 'function',
    '_id.buffer.writeUInt16BE': 'function',
    '_id.buffer.writeUintLE': 'function',
    '_id.buffer.writeUint32LE': 'function',
    '_id.buffer.writeUint16LE': 'function',
    '_id.buffer.writeUint8': 'function',
    '_id.buffer.writeUintBE': 'function',
    '_id.buffer.writeUint32BE': 'function',
    '_id.buffer.writeUint16BE': 'function',
    '_id.buffer.writeIntLE': 'function',
    '_id.buffer.writeInt32LE': 'function',
    '_id.buffer.writeInt16LE': 'function',
    '_id.buffer.writeInt8': 'function',
    '_id.buffer.writeIntBE': 'function',
    '_id.buffer.writeInt32BE': 'function',
    '_id.buffer.writeInt16BE': 'function',
    '_id.buffer.readFloatLE': 'function',
    '_id.buffer.readFloatBE': 'function',
    '_id.buffer.readDoubleLE': 'function',
    '_id.buffer.readDoubleBE': 'function',
    '_id.buffer.writeFloatLE': 'function',
    '_id.buffer.writeFloatBE': 'function',
    '_id.buffer.writeDoubleLE': 'function',
    '_id.buffer.writeDoubleBE': 'function',
    '_id.buffer.asciiSlice': 'function',
    '_id.buffer.base64Slice': 'function',
    '_id.buffer.base64urlSlice': 'function',
    '_id.buffer.latin1Slice': 'function',
    '_id.buffer.hexSlice': 'function',
    '_id.buffer.ucs2Slice': 'function',
    '_id.buffer.utf8Slice': 'function',
    '_id.buffer.asciiWrite': 'function',
    '_id.buffer.base64Write': 'function',
    '_id.buffer.base64urlWrite': 'function',
    '_id.buffer.latin1Write': 'function',
    '_id.buffer.hexWrite': 'function',
    '_id.buffer.ucs2Write': 'function',
    '_id.buffer.utf8Write': 'function',
    '_id.buffer.parent': 'object',
    '_id.buffer.offset': 'number',
    '_id.buffer.copy': 'function',
    '_id.buffer.toString': 'function',
    '_id.buffer.equals': 'function',
    '_id.buffer.inspect': 'function',
    '_id.buffer.compare': 'function',
    '_id.buffer.indexOf': 'function',
    '_id.buffer.lastIndexOf': 'function',
    '_id.buffer.includes': 'function',
    '_id.buffer.fill': 'function',
    '_id.buffer.write': 'function',
    '_id.buffer.toJSON': 'function',
    '_id.buffer.subarray': 'function',
    '_id.buffer.slice': 'function',
    '_id.buffer.swap16': 'function',
    '_id.buffer.swap32': 'function',
    '_id.buffer.swap64': 'function',
    '_id.buffer.toLocaleString': 'function',
    '_id.serverVersions': 'array',
    '_id.platforms': 'array',
    '_id.topologies': 'array',
    '_id.help': 'function',
    name: 'string',
    email: 'string',
    password: 'string'
  }
}
# Коллекции:
<img width="986" height="586" alt="comments" src="https://github.com/user-attachments/assets/6bbc7671-31db-476e-9526-1ba6834293d2" />
<img width="957" height="503" alt="users" src="https://github.com/user-attachments/assets/6a4dac45-4880-45e3-b5a2-1ffeeca78cb6" />
<img width="782" height="492" alt="theaters" src="https://github.com/user-attachments/assets/43628ceb-8003-4c45-a53d-25ba6ef6d430" />
<img width="974" height="513" alt="sessions" src="https://github.com/user-attachments/assets/3c05b8a1-fa82-4e64-9124-d3a01380fd23" />
<img width="1006" height="646" alt="movies" src="https://github.com/user-attachments/assets/e3492b45-5def-42f1-a728-82b6275abeec" />

# Postgre:
<img width="396" height="293" alt="image" src="https://github.com/user-attachments/assets/e953b7dc-f169-4a9f-8cb4-d5f4bf246d88" />

SELECT table_name, column_name, data_type

"restored_comments"	"_id"	"character varying"  
"restored_comments"	"name"	"text"  
"restored_comments"	"email"	"text"  
"restored_comments"	"movie_id"	"text"  
"restored_comments"	"text"	"text"  
"restored_comments"	"date"	"text"  
"restored_movies"	"_id"	"character varying"  
"restored_movies"	"plot"	"text"  
"restored_movies"	"runtime"	"bigint"  
"restored_movies"	"num_mflix_comments"	"bigint"  
"restored_movies"	"title"	"text"  
"restored_movies"	"fullplot"	"text"  
"restored_movies"	"released"	"text"  
"restored_movies"	"rated"	"text"  
"restored_movies"	"awards_wins"	"bigint"  
"restored_movies"	"awards_nominations"	"bigint"  
"restored_movies"	"awards_text"	"text"  
"restored_movies"	"lastupdated"	"text"  
"restored_movies"	"year"	"bigint"  
"restored_movies"	"imdb_rating"	"double precision"  
"restored_movies"	"imdb_votes"	"bigint"  
"restored_movies"	"imdb_id"	"bigint"  
"restored_movies"	"type"	"text"  
"restored_movies"	"tomatoes_viewer_rating"	"bigint"  
"restored_movies"	"tomatoes_viewer_numReviews"	"bigint"  
"restored_movies"	"tomatoes_viewer_meter"	"bigint"  
"restored_movies"	"tomatoes_lastUpdated"	"text"  
"restored_movies"	"poster"	"text"  
"restored_movies"	"tomatoes_fresh"	"bigint"  
"restored_movies"	"tomatoes_critic_rating"	"double precision"  
"restored_movies"	"tomatoes_critic_numReviews"	"bigint"   
"restored_movies"	"tomatoes_critic_meter"	"bigint"  
"restored_movies"	"tomatoes_rotten"	"bigint"  
"restored_movies"	"tomatoes_dvd"	"text"  
"restored_movies"	"tomatoes_website"	"text"  
"restored_movies"	"tomatoes_production"	"text"  
"restored_movies"	"tomatoes_consensus"	"text"  
"restored_movies"	"tomatoes_boxOffice"	"text"  
"restored_movies"	"metacritic"	"bigint"  
"restored_movies_cast"	"parent_uuid"	"character varying"  
"restored_movies_cast"	"target_val"	"text"  
"restored_movies_countries"	"parent_uuid"	"character varying"  
"restored_movies_countries"	"target_val"	"text"  
"restored_movies_directors"	"parent_uuid"	"character varying"  
"restored_movies_directors"	"target_val"	"text"  
"restored_movies_genres"	"parent_uuid"	"character varying"  
"restored_movies_genres"	"target_val"	"text"  
"restored_movies_languages"	"parent_uuid"	"character varying"  
"restored_movies_languages"	"target_val"	"text"  
"restored_movies_writers"	"parent_uuid"	"character varying"  
"restored_movies_writers"	"target_val"	"text"  
"restored_sessions"	"_id"	"character varying"  
"restored_sessions"	"user_id"	"text"  
"restored_sessions"	"jwt"	"text"  
"restored_theaters"	"_id"	"character varying"  
"restored_theaters"	"theaterId"	"bigint"  
"restored_theaters"	"location_address_street1"	"text"  
"restored_theaters"	"location_address_city"	"text"  
"restored_theaters"	"location_address_state"	"text"  
"restored_theaters"	"location_address_zipcode"	"text"  
"restored_theaters"	"location_geo_type"	"text"  
"restored_theaters"	"location_geo_coordinates_lon"	"double precision"  
"restored_theaters"	"location_geo_coordinates_lat"	"double precision"  
"restored_theaters"	"location_address_street2"	"text"  
"restored_users"	"_id"	"character varying"  
"restored_users"	"name"	"text"  
"restored_users"	"email"	"text"  
"restored_users"	"password"	"text"  

## Рез-ты работы SQL to Mongo
<img width="1380" height="693" alt="image" src="https://github.com/user-attachments/assets/096d41c8-4750-43a5-a7d5-95703a7fb7f9" />
# SQL:
<img width="414" height="297" alt="image" src="https://github.com/user-attachments/assets/c1af2a68-6f2e-4650-b41c-78f083ebb461" />

SELECT table_name, column_name, data_type

"album"	"album_id"	"integer"
"album"	"title"	"character varying"
"album"	"artist_id"	"integer"
"artist"	"artist_id"	"integer"
"artist"	"name"	"character varying"
"customer"	"customer_id"	"integer"
"customer"	"first_name"	"character varying"
"customer"	"last_name"	"character varying"
"customer"	"company"	"character varying"
"customer"	"address"	"character varying"
"customer"	"city"	"character varying"
"customer"	"state"	"character varying"
"customer"	"country"	"character varying"
"customer"	"postal_code"	"character varying"
"customer"	"phone"	"character varying"
"customer"	"fax"	"character varying"
"customer"	"email"	"character varying"
"customer"	"support_rep_id"	"integer"
"employee"	"employee_id"	"integer"
"employee"	"last_name"	"character varying"
"employee"	"first_name"	"character varying"
"employee"	"title"	"character varying"
"employee"	"reports_to"	"integer"
"employee"	"birth_date"	"timestamp without time zone"
"employee"	"hire_date"	"timestamp without time zone"
"employee"	"address"	"character varying"
"employee"	"city"	"character varying"
"employee"	"state"	"character varying"
"employee"	"country"	"character varying"
"employee"	"postal_code"	"character varying"
"employee"	"phone"	"character varying"
"employee"	"fax"	"character varying"
"employee"	"email"	"character varying"
"genre"	"genre_id"	"integer"
"genre"	"name"	"character varying"
"invoice"	"invoice_id"	"integer"
"invoice"	"customer_id"	"integer"
"invoice"	"invoice_date"	"timestamp without time zone"
"invoice"	"billing_address"	"character varying"
"invoice"	"billing_city"	"character varying"
"invoice"	"billing_state"	"character varying"
"invoice"	"billing_country"	"character varying"
"invoice"	"billing_postal_code"	"character varying"
"invoice"	"total"	"numeric"
"invoice_line"	"invoice_line_id"	"integer"
"invoice_line"	"invoice_id"	"integer"
"invoice_line"	"track_id"	"integer"
"invoice_line"	"unit_price"	"numeric"
"invoice_line"	"quantity"	"integer"
"media_type"	"media_type_id"	"integer"
"media_type"	"name"	"character varying"
"playlist"	"playlist_id"	"integer"
"playlist"	"name"	"character varying"
"playlist_track"	"playlist_id"	"integer"
"playlist_track"	"track_id"	"integer"
"track"	"track_id"	"integer"
"track"	"name"	"character varying"
"track"	"album_id"	"integer"
"track"	"media_type_id"	"integer"
"track"	"genre_id"	"integer"
"track"	"composer"	"character varying"
"track"	"milliseconds"	"integer"
"track"	"bytes"	"integer"
"track"	"unit_price"	"numeric"

# Mongo:
<img width="270" height="198" alt="image" src="https://github.com/user-attachments/assets/f4b4bfe5-d64a-445c-96bc-5628248f976d" />
Коллекции:
<img width="852" height="581" alt="image" src="https://github.com/user-attachments/assets/b9ffa375-bc26-4747-ad36-36a02faf4105" />
<img width="906" height="735" alt="image" src="https://github.com/user-attachments/assets/e09504d4-6a89-4633-b840-307eab54717d" />
<img width="767" height="356" alt="image" src="https://github.com/user-attachments/assets/4158aef3-ac2b-4327-97bf-9c1f458d8892" />
<img width="832" height="290" alt="image" src="https://github.com/user-attachments/assets/7d3ba47a-9413-41c1-83e4-352502a6d582" />
<img width="778" height="282" alt="image" src="https://github.com/user-attachments/assets/7e25d34e-615e-4b2a-b104-fc525bc48901" />
<img width="876" height="632" alt="image" src="https://github.com/user-attachments/assets/71bb250c-f424-43b6-98d7-48fecfb44488" />

{
  restored_artist: {
    _id: 'number',
    artist_id: 'number',
    name: 'string',
    global_uuid: 'string',
    nested_album: 'array',
    'nested_album[].album_id': 'number',
    'nested_album[].title': 'string',
    'nested_album[].artist_id': 'number',
    'nested_album[].nested_track': 'array',
    'nested_album[].nested_track[].track_id': 'number',
    'nested_album[].nested_track[].name': 'string',
    'nested_album[].nested_track[].album_id': 'number',
    'nested_album[].nested_track[].media_type_id': 'number',
    'nested_album[].nested_track[].genre_id': 'number',
    'nested_album[].nested_track[].composer': 'string',
    'nested_album[].nested_track[].milliseconds': 'number',
    'nested_album[].nested_track[].bytes': 'number',
    'nested_album[].nested_track[].unit_price': 'number',
    'nested_album[].nested_track[].playlist_details': 'array',
    'nested_album[].nested_track[].playlist_details[].playlist_id': 'number'
  },
  restored_employee: {
    _id: 'number',
    employee_id: 'number',
    last_name: 'string',
    first_name: 'string',
    title: 'string',
    reports_to: 'null',
    birth_date: 'string',
    hire_date: 'string',
    address: 'string',
    city: 'string',
    state: 'string',
    country: 'string',
    postal_code: 'string',
    phone: 'string',
    fax: 'string',
    email: 'string',
    global_uuid: 'string'
  },
  restored_customer: {
    _id: 'number',
    customer_id: 'number',
    first_name: 'string',
    last_name: 'string',
    company: 'string',
    address: 'string',
    city: 'string',
    state: 'string',
    country: 'string',
    postal_code: 'string',
    phone: 'string',
    fax: 'string',
    email: 'string',
    support_rep_id: 'number',
    global_uuid: 'string',
    nested_invoice: 'array',
    'nested_invoice[].invoice_id': 'number',
    'nested_invoice[].customer_id': 'number',
    'nested_invoice[].invoice_date': 'string',
    'nested_invoice[].billing_address': 'string',
    'nested_invoice[].billing_city': 'string',
    'nested_invoice[].billing_state': 'string',
    'nested_invoice[].billing_country': 'string',
    'nested_invoice[].billing_postal_code': 'string',
    'nested_invoice[].total': 'number',
    'nested_invoice[].nested_invoice_line': 'array',
    'nested_invoice[].nested_invoice_line[].invoice_line_id': 'number',
    'nested_invoice[].nested_invoice_line[].invoice_id': 'number',
    'nested_invoice[].nested_invoice_line[].track_id': 'number',
    'nested_invoice[].nested_invoice_line[].unit_price': 'number',
    'nested_invoice[].nested_invoice_line[].quantity': 'number'
  },
  restored_genre: {
    _id: 'number',
    genre_id: 'number',
    name: 'string',
    global_uuid: 'string'
  },
  restored_media_type: {
    _id: 'number',
    media_type_id: 'number',
    name: 'string',
    global_uuid: 'string'
  },
  restored_playlist: {
    _id: 'number',
    playlist_id: 'number',
    name: 'string',
    global_uuid: 'string',
    track_details: 'array',
    'track_details[].track_id': 'number'
  }
}






