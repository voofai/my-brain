---
date: 2023-08-07
share: true
title: Ленивые коллекции и прокси в Hibernate
tags:
  - java
  - jpa
hide:
  - navigation
  - toc
---

Для примера представим, что мы делаем многопользовательскую игру и у нас есть класс Гильдия (`Guild`) и связанные с ней отношением один ко многим список игроков (`Player`), состоящих в этой гильдии.

```java
@Entity
public class Guild {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "guild")
    private List<Player> members = new ArrayList<>();
}

@Entity
public class Player {
    @Id
    @GeneratedValue
    private Long id;
    private Integer level;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "guild_id")
    private Guild guild;
}
```

## Прокси классы

В отличие от `OneToMany` отношение `ManyToOne` по умолчанию не является ленивым. Сами разработчики Hibernate признают это решение ошибочным и рекомендуют вообще все связи между сущностями делать ленивыми. Для этого мы добавляем свойство `fetch = FetchType.LAZY` в аннотацию `@ManyToOne`.

При получении из БД объекта `Player` связанный с ним объект `guild` не загружается сразу, а будет загружен только при первом обращении к нему.

```java
Player player = playerRepository.findById(playerId); // загружаем только player, guild не загружается
String guildName = player.getGuild().getName(); // получение guild из БД отдельным запросом
```

По умолчанию это реализовано в Hibernate через создание прокси класса наследника `Guild`. Hibernate передает в прокси объект идентификатор сущности (в нашем случае это `guild_id`) и записывает в поле `guild`.

Поскольку идентификатор уже хранится в прокси `guild` вызов метода `getId()` в отличие от `getName()` не приведет к запросу в БД:

```java
Player player = playerRepository.findById(playerId); // загружаем только player, guild не загружается
Long guildId = player.getGuild().getId(); // не приводит к запросу в таблицу guild
```

Зная `id` сущности можно самостоятельно создать прокси через вызов метода `EntityManager.getReference()` (или `getReferenceById` из Spring Data JPA). Это удобно если мы хотим связать игрока с гильдией, но не хотим для этого загружать весь объект `Guild` из БД:

```java
Player player = playerRepository.findById(playerId);
Guild guild = guildRepository.getReferenceById(guildId); // оборачиваем guildId в прокси без обращения к БД
player.setGuild(guild); // делаем update записи в таблице player
```

Запустив этот код видим, что обращений к таблице `guild` не происходит:

```sql
select id, guild_id, level from player where id = ?
update player set guild_id = 1, level = 99 where id = ?
```

Также использование прокси может быть полезным, если мы еще не знаем понадобится или нет нам данные из сущности. Например, перед добавлением игрока в гильдию мы проверяем есть ли там свободные места:

```java
public void joinToGuild(Guild guild, Player player) {
    if (!guild.isFull()) {
        player.setGuild(guild);
    }
}
```

Передав в функцию прокси объект `Player` вместо загруженного из БД мы сэкономим обращение к таблице `player` если в гильдии нет для него места.

## Ленивые коллекции

Перейдем к классу `Guild` и посмотрим, что мы можем сделать с ленивой коллекцией `members` не загружая ее из БД целиком.

```java
@Entity
public class Guild {
    // ...

    @OneToMany(mappedBy = "guild")
    private List<Player> members = new ArrayList<>();
}
```

Отношение `@OneToMany` по умолчанию является ленивым, а значит при запросе объекта `Guild` список `members` будет загружен из БД не сразу, а только при первом обращении к нему. Это поведение заложено в классе под названием `PersistentBag`, наследнике `List`. Его Hibernate запишет вместо стандартного `ArrayList` в поле `members` при загрузке сущности.

Для примера, если мы просто запросим сущность `Guild`, то связанный список игроков загружен не будет:

```java
Guild guild = guildRepository.findById(guildId); // выборка только из таблицы guild, список members не загружается
```

Если же мы попытаемся, например, получить количество участников гильдии, используя метод `members.size()`, то это уже приведет к загрузке всех членов гильдии из БД:

```java
Integer guildSize = guild.getMembers().size(); // select id, level, guild_id from player where guild_id = ?
```

Но, используя утилитный класс `org.hibernate.Hibernate`, мы можем получить размер ленивой коллекции без ее полной загрузки:

```java
Guild guild = guildRepository.findById(guildId);
Integer guildSize = Hibernate.size(guild.getMembers());
```

В данном примере Hibernate сгенерирует SQL запрос с `count`:

```sql
select id, name from guild where id = ?
select count(id) from player where guild_id = ?
```

Еще один полезный метод - `Hibernate.contains()`, проверяет, существует ли сущность в ленивом списке.
Например, у нас есть id игрока и мы хотим проверить, состоит ли он в гильдии:

```java
Guild guild = guildRepository.findById(guildId); // получаем guild из БД без загрузки списка members
Player player = playerRepository.getReferenceById(playerId); // получаем прокси из playerId без обращения к БД
boolean isMemberOf = Hibernate.contains(guild.getMembers(), player); // делаем поиск игрока по id без загрузки members или player
```

Код выше приведет только к двум SQL запросам:

```sql
select id, name from guild where id = ?
select 1 from player where guild_id = ? and id = ?
```

Вообще говоря, не все методы ленивых коллекций приводят к полной загрузке содержимого. Например, этого не происходит при добавлении нового элемента в `List` через вызов функции `add` или `addAll`:

Для примера добавим в класс игрока список его достижений (`achievements`) с ленивой загрузкой:

```java
@Entity
public class Player {
    @Id
    @GeneratedValue
    private Long id;
    private Integer level;

    @OneToMany(mappedBy = "player", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Achievement> achievements = new ArrayList<>();
}
```

Попробуем добавить новое достижение игроку:

```java
Player player = playerRepository.findById(playerId); // запрашиваем Player из БД без загрузки списка achievements
player.getAchievements().add(new Achievement("Гроза гоблинов", player)); // не происходит загрузки списка achievements 
```

По SQL запросам видим, что полной загрузки списка `achievements` не происходит:

```sql
select id, guild_id, level from player where id = ?
select next value for achievement_seq
insert into achievement (name, player_id, id) values ("Гроза гоблинов", 1, 1)
```

Но это работает только для коллекций типа `List`. Если же мы заменим тип `achievements` на `Set`, то `Hibernate` будет вынужден загрузить из БД всю коллекцию при вызове `add`, чтобы проверить уникальность нового элемента.