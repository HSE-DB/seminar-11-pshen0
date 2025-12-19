## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on test\_cluster  \(cost=59.17..7696.73 rows=5000 width=68\) \(actual time=17.334..72.732 rows=500102 loops=1\) |
|   Recheck Cond: \(category = 'A'::text\) |
|   Heap Blocks: exact=8334 |
|   -&gt;  Bitmap Index Scan on test\_cluster\_cat\_idx  \(cost=0.00..57.92 rows=5000 width=0\) \(actual time=16.260..16.260 rows=500102 loops=1\) |
|         Index Cond: \(category = 'A'::text\) |
| Planning Time: 0.215 ms |
| Execution Time: 83.266 ms |
    
    *Объясните результат:*
До кластеризации строки с категорией A физически разбросаны по всей таблице, поэтому индекс возвращает большое количество TID’ов, а дальше выполняется Bitmap Heap Scan с чтением тысяч heap-блоков. Bitmap используется, потому что совпадений очень много, и точечный Index Scan был бы слишком дорогим. Основное время уходит на случайное чтение страниц таблицы.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
[2025-12-19 22:45:17] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx

[2025-12-19 22:45:17] completed in 412 ms

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
| QUERY PLAN |
| :--- |
| Index Scan using test\_cluster\_cat\_idx on test\_cluster  \(cost=0.42..14587.00 rows=498833 width=39\) \(actual time=0.067..54.753 rows=500102 loops=1\) |
|   Index Cond: \(category = 'A'::text\) |
| Planning Time: 0.316 ms |
| Execution Time: 68.351 ms |
    
    *Объясните результат:*
После кластеризации строки с одинаковой категорией лежат рядом на диске, поэтому планировщик выбирает обычный Index Scan. Чтение становится более последовательным, уменьшается количество обращений к разным heap-блокам, и пропадает необходимость в bitmap-сканировании. За счёт лучшей локальности данных общее время выполнения снижается.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
После кластеризации запрос стал заметно быстрее: время выполнения сократилось примерно с 83 мс до 68 мс. Выигрыш достигается за счёт улучшенной локальности данных и более эффективного доступа к таблице. При выборках большого процента строк кластеризация по фильтруемому столбцу действительно даёт практическую пользу.