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
   | QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=12.00..16.01 rows=1 width=33\) \(actual time=0.013..0.013 rows=0 loops=1\) |
|   Recheck Cond: \(category IS NULL\) |
|   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.010..0.010 rows=0 loops=1\) |
|         Index Cond: \(category IS NULL\) |
| Planning Time: 0.309 ms |
| Execution Time: 0.029 ms |
   
   *Объясните результат:*
BRIN хранит диапазоны значений по страницам.
Оптимизатор использует Bitmap Index Scan, чтобы отметить потенциально подходящие страницы, но всё равно делает Recheck, потому что BRIN не указывает точные строки.
Фактических строк с NULL нет → результат пустой, но индекс всё равно помогает быстро отфильтровать страницы.

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
   | QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=12.00..16.02 rows=1 width=33\) \(actual time=17.724..17.725 rows=0 loops=1\) |
|   Recheck Cond: \(\(category\)::text = 'INDEX'::text\) |
|   Rows Removed by Index Recheck: 150000 |
|   Filter: \(\(author\)::text = 'SYSTEM'::text\) |
|   Heap Blocks: lossy=1225 |
|   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.074..0.074 rows=12250 loops=1\) |
|         Index Cond: \(\(category\)::text = 'INDEX'::text\) |
| Planning Time: 0.245 ms |
| Execution Time: 17.767 ms |
   
   *Объясните результат (обратите внимание на bitmap scan):*
Используется bitmap-сканирование:
BRIN по category возвращает много страниц (12250 bitmap-элементов)
Индекс lossy -> PostgreSQL проверяет строки повторно
Фильтр по author применяется уже после чтения heap
Итог: BRIN плохо подходит для точечных условий + AND, если данные не коррелированы.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
| QUERY PLAN |
| :--- |
| Sort  \(cost=3100.11..3100.12 rows=5 width=7\) \(actual time=26.964..26.965 rows=6 loops=1\) |
|   Sort Key: category |
|   Sort Method: quicksort  Memory: 25kB |
|   -&gt;  HashAggregate  \(cost=3100.00..3100.05 rows=5 width=7\) \(actual time=26.934..26.935 rows=6 loops=1\) |
|         Group Key: category |
|         Batches: 1  Memory Usage: 24kB |
|         -&gt;  Seq Scan on t\_books  \(cost=0.00..2725.00 rows=150000 width=7\) \(actual time=0.019..7.769 rows=150000 loops=1\) |
| Planning Time: 0.113 ms |
| Execution Time: 27.034 ms |
   
   *Объясните результат:*
BRIN-индекс тут бесполезен, так как DISTINCT + ORDER BY требует пройти по всем данным. Планировщик выбирает последовательное сканирование, затем HashAggregate для уникальных значений и финальную сортировку. Малое число уникальных категорий снижает нагрузку на память, но чтение всей таблицы неизбежно.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
| QUERY PLAN |
| :--- |
| Aggregate  \(cost=3100.03..3100.05 rows=1 width=8\) \(actual time=13.564..13.565 rows=1 loops=1\) |
|   -&gt;  Seq Scan on t\_books  \(cost=0.00..3100.00 rows=14 width=0\) \(actual time=13.559..13.559 rows=0 loops=1\) |
|         Filter: \(\(author\)::text \~\~ 'S%'::text\) |
|         Rows Removed by Filter: 150000 |
| Planning Time: 0.191 ms |
| Execution Time: 13.585 ms |
   
   *Объясните результат:*
Запрос использует LIKE 'S%', но BRIN-индекс по author не помогает, так как данные распределены равномерно и префиксный поиск не позволяет эффективно отсеять диапазоны страниц. В результате выполняется Seq Scan с фильтрацией, и почти все строки отбрасываются.

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
| QUERY PLAN |
| :--- |
| Aggregate  \(cost=3476.88..3476.89 rows=1 width=8\) \(actual time=27.335..27.336 rows=1 loops=1\) |
|   -&gt;  Seq Scan on t\_books  \(cost=0.00..3475.00 rows=750 width=0\) \(actual time=27.329..27.330 rows=1 loops=1\) |
|         Filter: \(lower\(\(title\)::text\) \~\~ 'o%'::text\) |
|         Rows Removed by Filter: 149999 |
| Planning Time: 0.252 ms |
| Execution Time: 27.353 ms |
   
   *Объясните результат:*
Несмотря на функциональный индекс по LOWER(title), планировщик выбирает Seq Scan, потому что селективность запроса крайне низкая — почти все строки не подходят под условие. Использование индекса было бы дороже, чем последовательный проход по таблице.

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
| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=12.00..16.02 rows=1 width=33\) \(actual time=1.716..1.716 rows=0 loops=1\) |
|   Recheck Cond: \(\(\(category\)::text = 'INDEX'::text\) AND \(\(author\)::text = 'SYSTEM'::text\)\) |
|   Rows Removed by Index Recheck: 8831 |
|   Heap Blocks: lossy=73 |
|   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_auth\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.033..0.033 rows=730 loops=1\) |
|         Index Cond: \(\(\(category\)::text = 'INDEX'::text\) AND \(\(author\)::text = 'SYSTEM'::text\)\) |
| Planning Time: 0.277 ms |
| Execution Time: 1.744 ms |
   
   *Объясните результат:*
Составной BRIN-индекс резко сокращает количество подходящих страниц, так как учитывает корреляцию category и author. Bitmap становится менее «грязным», число lossy блоков и повторных проверок падает, из-за чего время выполнения запроса уменьшается почти на порядок. Это хороший пример того, как составной BRIN работает на связанных данных.