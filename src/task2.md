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
| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on t\_books  \(cost=21.03..1336.08 rows=750 width=33\) \(actual time=0.030..0.030 rows=1 loops=1\) |
|   Recheck Cond: \(to\_tsvector\('english'::regconfig, \(title\)::text\) @@ '''expert'''::tsquery\) |
|   Heap Blocks: exact=1 |
|   -&gt;  Bitmap Index Scan on t\_books\_fts\_idx  \(cost=0.00..20.84 rows=750 width=0\) \(actual time=0.021..0.021 rows=1 loops=1\) |
|         Index Cond: \(to\_tsvector\('english'::regconfig, \(title\)::text\) @@ '''expert'''::tsquery\) |
| Planning Time: 0.923 ms |
| Execution Time: 0.062 ms |
    
    *Объясните результат:*
Используется GIN-индекс по to_tsvector, поэтому планировщик сразу находит подходящие документы через Bitmap Index Scan. Затем выполняется Bitmap Heap Scan для точной проверки строк. Найдена всего одна запись, чтение затронуло один heap-блок, поэтому время выполнения минимальное — типичное и правильное поведение полнотекстового поиска с GIN.

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
| QUERY PLAN |
| :--- |
| Index Scan using t\_lookup\_pk on t\_lookup  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.020..0.021 rows=1 loops=1\) |
|   Index Cond: \(\(item\_key\)::text = '0000000455'::text\) |
| Planning Time: 0.141 ms |
| Execution Time: 0.035 ms |
     
     *Объясните результат:*
Выполняется обычный Index Scan по первичному ключу, так как поиск идёт по точному значению уникального поля. Индекс быстро указывает на нужную строку, после чего происходит одно обращение к таблице. Низкая стоимость и малое время выполнения ожидаемы для point-lookup по B-tree.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
| QUERY PLAN |
| :--- |
| Index Scan using t\_lookup\_clustered\_pkey on t\_lookup\_clustered  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.167..0.170 rows=1 loops=1\) |
|   Index Cond: \(\(item\_key\)::text = '0000000455'::text\) |
| Planning Time: 0.146 ms |
| Execution Time: 0.194 ms |
     
     *Объясните результат:*
План аналогичен предыдущему, но фактическое время чуть больше, так как кластеризация влияет только на физический порядок строк, а не на скорость индексного поиска по одному значению. Для одиночного обращения разницы почти нет, а иногда кластеризованная таблица даже немного медленнее из-за кэша и физического расположения данных.

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
| QUERY PLAN |
| :--- |
| Index Scan using t\_lookup\_value\_idx on t\_lookup  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.049..0.049 rows=0 loops=1\) |
|   Index Cond: \(\(item\_value\)::text = 'T\_BOOKS'::text\) |
| Planning Time: 0.316 ms |
| Execution Time: 0.079 ms |
     
     *Объясните результат:*
Используется индекс по item_value, поэтому выполняется Index Scan, несмотря на то что строка не найдена. Индекс позволяет быстро убедиться в отсутствии значения без полного сканирования таблицы. Время выполнения остаётся низким, так как чтение ограничивается индексом.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
| QUERY PLAN |
| :--- |
| Index Scan using t\_lookup\_clustered\_value\_idx on t\_lookup\_clustered  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.040..0.041 rows=0 loops=1\) |
|   Index Cond: \(\(item\_value\)::text = 'T\_BOOKS'::text\) |
| Planning Time: 0.223 ms |
| Execution Time: 0.053 ms |
     
     *Объясните результат:*
План полностью повторяет предыдущий случай. Кластеризация по первичному ключу не даёт преимуществ для поиска по item_value, так как порядок строк в таблице не соответствует этому полю. Разница во времени минимальна и статистически незначима.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
Производительность поиска по значению в обычной и кластеризованной таблицах практически одинакова, потому что в обоих случаях используется B-tree индекс по item_value. Кластеризация по другому столбцу не влияет на данный запрос, так как доступ к данным осуществляется через индекс, а не последовательным чтением таблицы.