# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   
   > Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.011 rows=0 loops=1)
   > Recheck Cond: (category IS NULL)
   > ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.008..0.009 rows=0 loops=1)
   > Index Cond: (category IS NULL)
   > Planning Time: 0.287 ms
   > Execution Time: 0.026 ms
   
   *Объясните результат:*
   
   Для запроса с условием `category IS NULL` PostgreSQL использовал стратегию Bitmap Index Scan + Bitmap Heap Scan с BRIN-индексом.
   
   BRIN-индекс позволил отфильтровать диапазоны страниц, в которых могли находиться NULL-значения.
   
   Из-за приближённого характера BRIN потребовалась дополнительная проверка условий.
   
   Строк, удовлетворяющих условию, найдено не было. Использование BRIN позволило избежать полного последовательного сканирования таблицы.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   
   > Bitmap Heap Scan on t_books  (cost=12.16..2353.86 rows=1 width=33) (actual time=13.665..13.666 rows=0 loops=1)
   > Recheck Cond: ((category)::text = 'INDEX'::text)
   > Rows Removed by Index Recheck: 150000
   > Filter: ((author)::text = 'SYSTEM'::text)
   > Heap Blocks: lossy=1225
   > ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=74447 width=0) (actual time=0.105..0.105 rows=12250 loops=1)
   > Index Cond: ((category)::text = 'INDEX'::text)
   > Planning Time: 0.196 ms
   > Execution Time: 13.685 ms
   
   *Объясните результат (обратите внимание на bitmap scan):*
   
   В запросе с условиями по category и author PostgreSQL использовал только BRIN-индекс по колонке category. Индекс по author не был задействован, так как его использование не снижало бы объём обрабатываемых данных.

   Из-за низкой селективности условия был сформирован lossy bitmap, что привело к необходимости повторной проверки строк (Recheck Cond) и удалению большого количества строк после перепроверки.

   Условие по author было применено как обычный фильтр после чтения данных.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   
   > Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=20.319..20.320 rows=6 loops=1)
   > Sort Key: category
   > Sort Method: quicksort  Memory: 25kB
   > ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=20.303..20.304 rows=6 loops=1)
   > Group Key: category
   > Batches: 1  Memory Usage: 24kB
   > ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.009..5.894 rows=150000 loops=1)
   > Planning Time: 0.101 ms
   > Execution Time: 20.389 ms
   
   *Объясните результат:*
   
   Для запроса с DISTINCT и ORDER BY PostgreSQL выполнил последовательное сканирование таблицы, так как для получения уникальных значений требуется просмотр всех строк.

   BRIN-индекс не был использован, поскольку он не хранит информацию об уникальности и не поддерживает упорядоченное чтение. Уникальные значения были получены с помощью HashAggregate, после чего результат был отсортирован оператором Sort.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   
   > Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=11.277..11.278 rows=1 loops=1)
   > ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=11.273..11.273 rows=0 loops=1)
   > Filter: ((author)::text ~~ 'S%'::text)
   > Rows Removed by Filter: 150000
   > Planning Time: 0.511 ms
   > Execution Time: 11.307 ms
   
   *Объясните результат:*
   
   В запросе с условием author LIKE 'S%' PostgreSQL выполнил последовательное сканирование таблицы, так как для префиксного поиска по строке отсутствует подходящий индекс.
   
   BRIN-индекс не используется, поскольку он не поддерживает поиск по шаблону LIKE и обладает слишком низкой селективностью.
   
   Все строки таблицы были проверены фильтром, после чего выполнена агрегация COUNT(*).

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   
   > Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=23.871..23.872 rows=1 loops=1)
   > ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=23.864..23.865 rows=1 loops=1)
   > Filter: (lower((title)::text) ~~ 'o%'::text)
   > Rows Removed by Filter: 149999
   > Planning Time: 0.636 ms
   > Execution Time: 23.899 ms
   
   *Объясните результат:*
   
   Несмотря на наличие функционального индекса по выражению LOWER(title) PostgreSQL выполнил последовательное сканирование таблицы. Так произошло, потому что оператор LIKE зависит от правил сортировки и стандартный B-tree индекс не может быть использован для префиксного поиска без специального класса операторов text_pattern_ops.
   
   В результате фильтрация выполнялась на уровне строк, после чего была выполнена агрегация COUNT(*).

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   
   > Bitmap Heap Scan on t_books  (cost=12.16..2353.86 rows=1 width=33) (actual time=0.981..0.982 rows=0 loops=1)
   > Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   > Rows Removed by Index Recheck: 8873
   > Heap Blocks: lossy=73
   > ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=74447 width=0) (actual time=0.037..0.037 rows=730 loops=1)
   > Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   > Planning Time: 0.407 ms
   > Execution Time: 1.051 ms
   
   *Объясните результат:*
   
   При использовании составного BRIN-индекса по колонкам category и author PostgreSQL смог значительно точнее отфильтровать диапазоны страниц, удовлетворяющие обоим условиям запроса. В результате количество lossy-блоков и строк, удалённых при повторной проверке условий, существенно сократилось.
   
   Поэтому резко уменьшился объём обрабатываемых данных и произошло значительное ускорение выполнения запроса по сравнению с вариантом с двумя одиночными BRIN-индексами.