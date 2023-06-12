---
share: true
---

Способ организации бизнес логики в виде вызова процедур. Каждая процедура выполняется в отдельной транзакции и обрабатывает отдельный запрос от слоя представления. Бизнес логика внутри процедуры перемешана с вызовами к [[Статьи/Базы данных/Базы данных|базе данных]].
По ощущениям самый распространенный паттерн написания кода в индустрии.
Благодаря своей простоте хорошо подходит для задач с простой бизнес логикой или для [[Статьи/Архитектура/Архитектура Систем/ETL|ETL]] задач.

## Пример
```java
@RequaredArgConstructor
public class Hotel {
  private final HotelDao hotelDao;

  @Transactional
  public void bookRoom(int roomId) throws Exception {
    Room room = hotelDao.getById(roomId);

    if (room == null) {
      throw new Exception("Room number: " + rroomId + " does not exist");
    } else {
      if (room.isBooked()) {
        throw new Exception("Room already booked!");
      } else {
        room.setBooked(true);
        hotelDao.update(room);
      }
    }
  }
}
```