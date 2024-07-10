---
layout: post
title: "5 Simple Things Learnt Using Android Room"
---


{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

While developing my Android app
[One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker) I decided
to use [Room](https://developer.android.com/jetpack/androidx/releases/room) as a
local database. I hadn't used Room in a few years and in this article I'll share
five more or less basic things that I learnt while picking it up again. Here we
go!

### 1. Use autoGenerate with the @PrimaryKey as 0 or null

Data is stored in Room as
[Entities](https://developer.android.com/training/data-storage/room/defining-data).
All Entities need to have a
[PrimaryKey](https://developer.android.com/training/data-storage/room/defining-data#primary-key)

In One Rep Max Tracker I initially had a `MovementEntity` with `id` as the
`@PrimaryKey`.

```
@Entity
data class MovementEntity(
    @PrimaryKey val id: Long,
    val name: String,
)
```

To add `MovementEntity` to the database I used the `insert` method of
`MovementDao`.

```
@Dao
interface MovementDao {
    @Insert
    suspend fun insert(movement: MovementEntity): Long
}
```

This worked fine, but had two problems:

1. I needed to set the id of MovementEntity myself
2. If I tried inserting a MovementEntity with an id that already existed in
   the database the app would crash:
   `android.database.sqlite.SQLiteConstraintException: UNIQUE constraint failed: MovementEntity.id (code 1555 SQLITE_CONSTRAINT_PRIMARYKEY)`

Luckily, Room has
[autoGenerate](<https://developer.android.com/reference/kotlin/androidx/room/PrimaryKey#autoGenerate()>)
to generate the id automatically. Importantly, the value
[needs to be](<https://developer.android.com/reference/kotlin/androidx/room/PrimaryKey#autoGenerate()>)
either 0 or null for the id to be generated.

```
@Entity
data class MovementEntity(
    @PrimaryKey(autoGenerate = true) val id: Long, // Needs to be either 0 or null to automatically generate the id
    val name: String,
)
```

After adding `autoGenerate = true` and using 0 for the id both problems were
resolved.

## 2. Handling nullability

Not knowing have to handle nullability with Room caused my app to crash.

I had the following Query fetching all `OneRMEntities` with a certain `id`.
Since all `oneRMids` are unique this would return at most one `OneRMEntity`.

```
@Query("SELECT * FROM oneRMEntity WHERE oneRMid = :id")
fun getOneRM(id: Long): Flow<OneRMEntity>
```

In my repository I called the method and mapped it to an external model to be
displayed in the UI.

```
override fun getOneRM(id: Long): Flow<OneRMInfo> =
    oneRMDao.getOneRM(id).map { it.asExternalModel() }
```

However, this caused a crash whenever no `OneRMEntity` with the `id` was found.

```
java.lang.NullPointerException: Parameter specified as non-null is null: method OneRMEntityKt.asExternalModel, parameter <this>
at OneRMEntityKt.asExternalModel(Unknown Source:2)
```

After some research I found the solution. First, I changed the return type to
`OneRMEntity?` to correctly show that it can be null.

```
@Query("SELECT * FROM oneRMEntity WHERE oneRMid = :id")
fun getOneRM(id: Long): Flow<OneRMEntity?>
```

Then I added `filterNotNull()` before mapping it to my external model.

```
override fun getOneRM(id: Long): Flow<OneRMInfo> =
    oneRMDao.getOneRM(id).filterNotNull().map { it.asExternalModel() }
```

Voila, no more crashes!

## 3. How to combine data from two tables

Sometimes it's necessary to combine data from two tables in Room.

In my case I needed information from both `MovementEntity` and `OneRMEntity`. I
created this `@Query`:

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
fun getMovements(): Flow<Map<MovementEntity, List<OneRMEntity>>>
```

I used [LEFT JOIN](https://www.w3schools.com/sql/sql_join_left.asp) to get all
`MovementEntities` and only the matching `OneRMEntities`.

The mapping method `asExternalMovement()` looked like this:

```
fun Map.Entry<MovementEntity, List<OneRMEntity>>.asExternalMovement() = Movement(
    id = key.id,
    name = key.name,
    weight = value.maxByOrNull { it.weightInKilos }?.weightInKilos
)
```

The `Movement` gets the `id` and `name` from the `MovementEntity` and the
`weight` from the list of `OneRMEntities`. There are other ways to combine data
from two tables but this has worked well for me so far.

## 4. Use Flow for automatic updates

We often want the UI to always reflect the data in Room, meaning that the UI
should update whenever the data in Room changes. Initially, I solved this without [Flow](https://developer.android.com/kotlin/flow).

In the `MovementDao` I had the method `getMovements()` which returned `Map<MovementEntity, List<OneRMEntity>>`.

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
suspend fun getMovements(): Map<MovementEntity, List<OneRMEntity>>
```

I called `getMovements()` in the `MovementRepository` and mapped the result to a
list of `Movement`.

```
override suspend fun getMovements(): List<Movement> =
    movementDao.getMovements().entries.map {
        it.asExternalMovement()
    }
```

In the ViewModel I called the method in the repository and updated the
\_uiState.

```
fun getMovements() {
    viewModelScope.launch {
        val movements = Success(movementsRepository.getMovements())
        _uiState.update { movements }
    }
}
```

However, if I would update the database either by adding, editing or deleting a
`Movement`, I had to call `getMovements()` to fetch the latest state of the
database.

```
fun addMovement(movement: Movement) {
    viewModelScope.launch {
        val movementId = movementsRepository.setMovement(movement)
	    ...
        getMovements() // Had to do this to make sure the UI reflected the database
    }
}

fun editMovement(movement: Movement) {
    viewModelScope.launch {
        movementsRepository.setMovement(movement)
        getMovements() // Had to do this to make sure the UI reflected the database
    }
}

fun deleteMovement(id: Long) {
    viewModelScope.launch {
        movementsRepository.deleteMovement(id)
        getMovements() // Had to do this to make sure the UI reflected the database
    }
}
```

[Flow](https://medium.com/androiddevelopers/room-flow-273acffe5b57) can easily be used to accomplish the same thing in a much better way. Let's see the necessary changes.

In the Dao I removed the suspend keyword and changed the return type to
`Flow<Map<MovementEntity, List<OneRMEntity>>>`.

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
fun getMovements(): Flow<Map<MovementEntity, List<OneRMEntity>>>
```

In the repository I changed the return type.

```
override suspend fun getMovements(): Flow<List<Movement>> =
    movementDao.getMovements().map { map ->
        map.entries.map {
            it.asExternalMovement()
        }
    }
```

And in the ViewModel I used `.collect` to collect the Flow.

```
fun getMovements() {
    viewModelScope.launch {
        movementsRepository.getMovements()
            .collect { movements ->
                _uiState.update { Success(movements) }
            }
    }
}
```

That's it! Now the code was ready for reaping the benefits of Flow. I could
remove the call `getMovements()` in all other methods in the ViewModel.

```
fun addMovement(movement: Movement, weightUnit: String) {
    viewModelScope.launch {
        val movementId = movementsRepository.setMovement(movement)
        ...
    }
}

fun editMovement(movement: Movement) {
    viewModelScope.launch {
        movementsRepository.setMovement(movement)
    }
}

fun deleteMovement(id: Long) {
    viewModelScope.launch {
        movementsRepository.deleteMovement(id)
    }
}
```

Much better!

## 5. Debugging with App Inspection

Being able to debug well is crucial to being a good developer. Android Studio
has [great support](https://developer.android.com/studio/inspect/database) for
debugging Room. In the App Inspection window you can inspect, query, and modify your
app's databases while your app is running. This is extremely useful and greatly
speeds up development.

Here is an example of App Inspection of the database of One Rep Max Tracker.
![screenshot](/assets/images/blog/app-inspection.png){:width="800px"}

## Conclusion

Room is an extremely useful tool for Android development. In this article I've
presented five more or less basic things that I learnt while using Room for my app
[One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker).

What are some things about Room that you'd like to share? Let me know in the comments!
