# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   Seq Scan on t_books tb  (cost=0.00..3100.00 rows=1 width=33)
  Filter: ((title)::text = 'Oracle Core'::text)
   
   *Объясните результат:*
   Проводим поиск по не индексированному полю, соответственно, планировщик использует последовательный обход всей таблички, что выходит сильно
   дорого по костам

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   Не понимаю, какой нужно втсавить результат выполнения: у нас создалось два индекса

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*

|schemaname|tablename|indexname|indexdef|
|----------|---------|---------|--------|
|public|t_books|t_books_id_pk|CREATE UNIQUE INDEX t_books_id_pk ON public.t_books USING btree (book_id)|
|public|t_books|t_books_title_idx|CREATE INDEX t_books_title_idx ON public.t_books USING btree (title)|
|public|t_books|t_books_active_idx|CREATE INDEX t_books_active_idx ON public.t_books USING btree (is_active)|

   
   *Объясните результат:*
   всё работает

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   статистика обновилась

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*

   |QUERY PLAN|
|----------|
|Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.026..0.027 rows=1 loops=1)|
|  Index Cond: ((title)::text = 'Oracle Core'::text)|
|Planning Time: 0.068 ms|
|Execution Time: 0.042 ms|

   
   *Объясните результат:*
   Теперь оптимизатор выполняет запрос наиболее эффективным методом - поиск по индексному полю через индексное сканирование.
   Расходы на запрос снизились почти в 370 раз, фактическое время выполнения оказалось лучше планируемого.

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   

|Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.014..0.015 rows=1 loops=1)|
|  Index Cond: (book_id = 18)|
|Planning Time: 0.056 ms|
|Execution Time: 0.027 ms|


   
   *Объясните результат:*
   Здесь также было индексное сканирование, т.к. СУБД по умолчанию создала индекс по поле первичного ключа. Фактическое время выполнения оказалось в два раза ниже планируемого

1. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   
   

|Seq Scan on t_books  (cost=0.00..2725.00 rows=75710 width=33) (actual time=0.006..7.580 rows=75212 loops=1)|
|  Filter: is_active|
|  Rows Removed by Filter: 74788|
|Planning Time: 0.047 ms|
|Execution Time: 9.006 ms|

   
   *Объясните результат:*
   по непонятным причинам фактическое время выполнения в стопицот раз выше планируемого, несмотря на созданный ранее индекс по полю.

1. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   
|total_rows|unique_titles|unique_categories|unique_authors|
|----------|-------------|-----------------|--------------|
|150000|150000|6|1003|


2.  Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:* индексы удалены

3.  Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
```sql
    CREATE INDEX books_title_idx ON t_books(title);
    CREATE INDEX books_category_idx ON t_books(category);
    CREATE INDEX books_category_author_idx ON t_books(category, author);
```
    
    *Объясните ваше решение:*
    при фильтрации по одному полю создаём соответствующий индеск по одному полю, при фильтрации по двум полям - индекс на двух полях





4.  Протестируйте созданные индексы.
    
    *Результаты тестов:*
    [Вставьте планы выполнения для каждого случая]
    
    *Объясните результаты:*
    [Ваше объяснение]

5.  Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
в
    
    *План выполнения:*
    
    

|Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=42.968..42.969 rows=0 loops=1)|
|  Filter: ((title)::text ~~* 'Relational%'::text)|
|  Rows Removed by Filter: 150000|
|Planning Time: 0.134 ms|
|Execution Time: 42.987 ms|

    
    *Объясните результат:*
    Здесь такое большое время выполнения из-за использования оператора ILIKE и % внутри, что требует полного просмотра строк и игнорирует сам индекс

1.  Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));

2.  Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    

|Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=33) (actual time=25.620..25.621 rows=0 loops=1)|
|  Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)|
|  Rows Removed by Filter: 150000|
|Planning Time: 0.060 ms|
|Execution Time: 25.637 ms|

    
    *Объясните результат:*
    Исспользуем не регистрозависимый оператор и приводим все к верхнему регистру сразу в индексе, поэтому запрос выполняется быстрее, т.е. в два  раза лучше, чем было


3.  Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    

|Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=48.494..48.498 rows=1 loops=1)|
|  Filter: ((title)::text ~~* '%Core%'::text)|
|  Rows Removed by Filter: 149999|
|Planning Time: 0.205 ms|
|Execution Time: 48.525 ms|

    
    *Объясните результат:*
    Использования % и регистрозависимого оператора ILIKE  игнорирует индекс и замедляет запрос

4.  Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    
    SQL Error [2BP01]: ERROR: cannot drop index t_books_id_pk because constraint t_books_id_pk on table t_books requires it
  Подсказка: You can drop constraint t_books_id_pk on table t_books instead.
  Где: SQL statement "DROP INDEX t_books_id_pk"
PL/pgSQL function inline_code_block line 9 at EXECUTE


Позиция ошибки:
    
    *Объясните результат:*
    Нельзя удалить индекс, связанный с полем первичного ключа

5.  Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    [Вставьте планы выполнения для обоих вариантов]
    
    *Объясните результаты:*
    [Ваше объяснение]

6.  Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    

|Index Scan using books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.102..0.103 rows=1 loops=1)|
|  Index Cond: ((title)::text = 'Oracle Core'::text)|
|Planning Time: 0.239 ms|
|Execution Time: 0.115 ms|

    
    *Объясните результат:*
    Оптимизатор выполняет запрос через индексное сканирование по созданному индексу

1.  Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    
    

|Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=41.547..41.548 rows=0 loops=1)|
|  Filter: ((title)::text ~~* 'Relational%'::text)|
|  Rows Removed by Filter: 150000|
|Planning Time: 0.093 ms|
|Execution Time: 41.562 ms|

    
    *Объясните результат:*
    Всё плоххо из-за использования ILIKE, оптимизатор игнорирует индекс и выполняет последовательное сканирование

1.  Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    [Вставьте ваш запрос]
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]