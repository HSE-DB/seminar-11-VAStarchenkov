## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    
    > Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.602..0.604 rows=1 loops=1)
    > Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    > Heap Blocks: exact=1
    > ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.588..0.588 rows=1 loops=1)
    > Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    > Planning Time: 0.439 ms
    > Execution Time: 0.637 ms
    
    *Объясните результат:*
    
    Для полнотекстового поиска PostgreSQL использовал GIN-индекс по выражению `to_tsvector('english', title)`. Индекс позволил быстро определить строки, содержащие искомый термин, после чего был сформирован bitmap страниц. Так как GIN является приближённым индексом, была выполнена повторная проверка условия (Recheck Cond) на уровне строк. В результате была прочитана всего одна страница таблицы, что обеспечило очень быстрое выполнение запроса.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     
     > Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.044..0.046 rows=1 loops=1)
     > Index Cond: ((item_key)::text = '0000000455'::text)
     > Planning Time: 0.412 ms
     > Execution Time: 0.068 ms
     
     *Объясните результат:*
     
     При поиске по первичному ключу PostgreSQL использовал обычный Index Scan по B-tree индексу, созданному автоматически для ограничения PRIMARY KEY.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     
     > Seq Scan on t_lookup_clustered  (cost=0.00..0.00 rows=1 width=256) (actual time=0.004..0.004 rows=0 loops=1)
     > Filter: ((item_key)::text = '0000000455'::text)
     > Planning Time: 0.339 ms
     > Execution Time: 0.015 ms
     
     *Объясните результат:*
     
     Несмотря на наличие первичного ключа, PostgreSQL выбрал последовательное сканирование таблицы t_lookup_clustered. Потому что таблица небольшая и полностью находится в памяти, а физическая кластеризация данных по ключу обеспечивает эффективный последовательный доступ. В данных условиях Seq Scan оказался дешевле, чем использование индексного доступа, что подтверждается меньшим временем выполнения запроса.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     
     > Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.035..0.036 rows=0 loops=1)
     > Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     > Planning Time: 0.264 ms
     > Execution Time: 0.049 ms
     
     *Объясните результат:*
     
     При поиске по значению item_value в обычной таблице PostgreSQL использовал индексное сканирование по вторичному B-tree индексу. Условие равенства высокоселективно, поэтому был выбран Index Scan, обеспечивающий быстрый доступ к одной строке.


18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     
     > Seq Scan on t_lookup_clustered  (cost=0.00..0.00 rows=1 width=256) (actual time=0.009..0.009 rows=0 loops=1)
     > Filter: ((item_value)::text = 'T_BOOKS'::text)
     > Planning Time: 0.717 ms
     > Execution Time: 0.029 ms
     
     *Объясните результат:*
     
     При поиске по колонке item_value в кластеризованной таблице PostgreSQL выбрал последовательное сканирование, несмотря на наличие индекса. Потому что таблица полностью находится в памяти, а физическая кластеризация выполнена по другой колонке (item_key) и не влияет на данный запрос.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     
     При поиске по значению item_value в обычной и кластеризованной таблицах существенного преимущества кластеризации не наблюдается. В обоих случаях PostgreSQL выбирает наиболее дешёвый план выполнения: индексное сканирование для обычной таблицы и последовательное сканирование для кластеризованной. Кластеризация влияет на производительность только в том случае, если физический порядок данных соответствует условиям запроса.