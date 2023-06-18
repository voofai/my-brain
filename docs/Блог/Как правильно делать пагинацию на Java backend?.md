---
date: 2023-06-18
share: true
hide:
  - navigation
  - toc
---

Пагинацией называется разделение большого массива данных на отдельные страницы для удобства использования. `Spring Data Jpa` для реализации пагинации предлагает использовать интерфейс `PagingAndSortingRepository`:

```java
public interface ProductRepository extends PagingAndSortingRepository<Product, Integer> {
    List<Product> findAll(Pageable pageable);
}
```

Но, стандартное решение от `Spring` подходит не всегда. Вызов метода `findAll` приведет к следующему SQL запросу:

```sql
SELECT * FROM product ORDER BY id LIMIT 10 OFFSET 20
```

Значения `LIMIT` и `OFFSET` будут выбраны в зависимости от того, какую по счету страницу запрашиваем. Этот подход называется **offset pagination**.
Проблема в том, что при использовании `OFFSET N` БД все равно выбирает все начальные строки запроса, и только после выборки пропускает `N` первых строк. И после страницы 20-30 это начинает негативно сказываться на скорости работы запроса. В этом случае стоит использовать подход **keyset/seek/cursor pagination)**. 

Суть подхода в том, что мы по некоторому полю (например по `id`) отсекаем в блоке `WHERE` уже показанные пользователю записи:

```sql
SELECT id FROM product WHERE id > 478 LIMIT 10
```

Данный подход сохраняет одинаковую скорость выборки не зависимо от номера страницы. Из минусов, не дает возможности сразу перейти на произвольную страницу.

## Как работать с большими ResultSet без пагинации?

По умолчанию многие JDBC драйверы (PostgreSQL, MySQL) загружают в память приложения весь результат запроса из БД. Но как быть, если нужно сделать выборку большого количества строк? Например, нам нужно сделать выгрузку таблицы на миллионы записей в файл на диске. Для этого мы можем настроить JDBC драйвер таким образом, чтобы он загружал данные небольшими порциям, пока мы последовательно проходим по ним курсором.
Как это сделать - лучше прочитать в документации на ваш JDBC драйвер, 100% универсального решения нет. 

В большинстве случаев достаточно установить параметр запроса `FetchSize`, например, используя `JdbcTemplate` из `Spring`:
```java
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
jdbcTemplate.setFetchSize(200);
jdbcTemplate.query("select * from product", rs -> );
  addRowToExcel(rs);
});
```

>[!info]
>В Oracle JDBC драйвере по умолчанию для всех запросов установлен `fetch size=10`. То есть, если вы запускаете `select`, который возвращает 50 строк, то потребуется 5 сетевых запросов от клиента в БД , чтобы выбрать все данные. Попробуйте установить больший fetch size, и, скорее всего, это положительно скажется на производительности.

## Как работать с большими ResultSet без пагинации в Spring Data JPA?

JPA для стриминга больших ResultSet из БД подходит плохо, слишком много способов отстрелить себе обе ноги, плюс плохая производительность. Но, если вы не ищите легких путей:

```java
public interface ProductRepository extends Repository<Product, Long> {
    
    @QueryHints(value = {
            @QueryHint(name = HINT_FETCH_SIZE, value = "50"),
            @QueryHint(name = HINT_CACHEABLE, value = "false"),
            @QueryHint(name = READ_ONLY, value = "true")
    })
    @Query("select p from Product p")
    Stream<Product> getAll();
}
```

Аннотациями `@QueryHint` мы говорим Hibernate, что не нужно помещать результаты в кэш второго уровня, результат получать из БД порциями по 50 записей и помечаем как `READ_ONLY` полученные объекты `Product`, чтобы избавить `Hibernate` от необходимости следить за изменениями в `Product`.

Но пачкой аннотаций история не заканчивается, важно еще правильно вызывать этот метод:

```java
  @Transactional(readOnly = true)
  public void processProducts() {
    try(Stream<Book> stream = productRepository.getAll()) {
      stream.forEach(product -> {
        addRowToExcel(product);
        entityManager.detach(product);
      });
    }
  }
```

Обратите внимание на строчку `entityManager.detach(product)`. `Hibernate` держит в кэше первого уровня все полученные в транзакции объекты. Без этой строчки на больших выборках приложение будет падать с `OutOfMemoryException`. Альтернативный и более производительный вариант каждые N итераций вызывать полную очистку контекста функцией `EntityManager.clean()`. Ну или все же не использовать JPA для этой задачи.

## Ссылки
- [OFFSET is bad for skipping previous rows](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)