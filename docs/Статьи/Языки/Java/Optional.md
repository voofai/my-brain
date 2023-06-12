---
share: true
---

`Optional` - контейнерный класс языка Java для представления значений, которых может не быть.
Если функция может вернуть `null`, то ее значение всегда нужно оборачивать в `Optional`. Это не только помогает убрать проблемы связанные с NPE, но и заставляет при моделировании более глубоко погружаться в предметную область. Задавать вопросы. В каких случаях может не быть этого значения? Что делать в этом месте, если значения нет?

При этом стоит придерживаться нескольких правил:

1. Не использовать `Optional` как аргумент в функциях, использовать `Optional` для возврата значения из функций.
2. Не использовать `Optional` тип для полей классов. Возвращать `Optional` в getter поля.
   ```java
    @Value
    class User {
      String firstName;
      Optional<String> middleName;
      String lastName;
    }
    ```
   Правильно:
   ```java
    @Value
    class User {
      String firstName;
      String middleName;
      String lastName;

      public Optional<String> getMiddleName() {
        return Optional.ofNullable(this.middleName);
      }
    }
    ```
1. Понимать разницу между `.orElse()` и `.orElseGet()`. 
   `orElse()` - используем, если значение уже посчитано 
   `.orElseGet()` - используем, если значение необходимо вычислить.
   Не правильно:
   ```java
    User user = users.stream().findFirst().orElse(new User("default", "1234"));
    ```
   Конструктор User будет вызван в любом случае, даже если значение Optional существует.
   Правильно:
   ```java
    User user = users.stream().findFirst().orElseGet(() -> new User("default", "1234"));
    ```
   Конструктор User будет вызван только если не нашли подходящего пользователя.

4. Не использовать связку `Optional.isPresent()` - `Optional.get()` там, где можно обойтись функцией `Optional.ifPresent()`.

5. Не увлекайтесь использованием Optional
   Не правильно:
   ```java
    return Optional.ofNullable(status).orElse("Not started yet.");
    ```
   Правильно:
   ```java
    return status == null ? "Not started yet." : status;
    ```