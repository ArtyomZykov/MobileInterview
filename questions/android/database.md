# Database

### MustHave

<details>
<summary>[bd-1] Что представляет собой Dao из Room?</summary>

Dao - это Data Access Object.

Аннотацией @Dao в Room помечаются интерфесы или абстрактные классы в которых описаны функции доступа к таблицам. Используя  @Insert, @Update, @Delete, @Query("SELECT * FROM RoomTournament WHERE id = :tournamentId LIMIT 1")

При кодогенерации, Room создаст реализацию этого интерфейса.
</details>

<details>
<summary>[bd-2] Какие данные нужно хранить в бд?
</summary>
[Наша статья](https://kmm.icerock.dev/university/data-storage/how-to-store-data)
</details>