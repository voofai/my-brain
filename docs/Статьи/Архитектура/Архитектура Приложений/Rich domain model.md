---
share: true
---

Богатая доменная модель это паттерн написания кода при котором объекты в программе содержат данные и код, который с ними работает. Благодаря этому мы можем не беспокоится, что бизнес логика расползется по всей системе. Является противоположностью [[Статьи/Архитектура/Архитектура Приложений/Anemic domain model|Anemic domain model]]

# Пример

```java
public class Order {
    private int totalSum;
    private List<OrderItem> items;

    public void addItem(OrderItem item) {
        items.add(item);
        totalSum = totalSum + item.getPrice();
    }

    public int getTotal() {
        return totalSum;
    }
}
```

# Когда применять?

Стоит использовать для модулей со сложной бизнес логикой, в противном случае скорее приведет к ухудшению читаемости кода. 

# Ссылки

- [Пример приложения](https://github.com/neherim/java-guild-katas/tree/master/money-transfer/money-transfer-rich)