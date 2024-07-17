---
layout: post
title: "5 Things I Learnt Using Android Room"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

While developing my Android app
[One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker) I decided
to use [Room](https://developer.android.com/jetpack/androidx/releases/room) as a
local database. I hadn't used Room in a few years and in this article I'll share
five basic things that I learnt while picking it up again. Here we
go!

## 1. AutoGenerate only works when @PrimaryKey is 0 or null

In Room, data is stored as
[Entities](https://developer.android.com/training/data-storage/room/defining-data) and all Entities need to have a
[PrimaryKey](https://developer.android.com/training/data-storage/room/defining-data#primary-key). PrimaryKeys can be generated automatically by using `@PrimaryKey(autoGenerate = true)`.

In One Rep Max Tracker I created a `MovementEntity` with `id` as the
PrimaryKey.

```
@Entity
data class MovementEntity(
    @PrimaryKey(autoGenerate = true) val id: Long,
    val name: String,
)
```

However, for some reason, the ids weren't being automatically generated. After a while I realised it was because I was setting the id to -1. The id *needs to be
either 0 or null* to be [generated](<https://developer.android.com/reference/kotlin/androidx/room/PrimaryKey#autoGenerate()>).

After updating my code the ids started being generated automatically as expected.

## 2. Handling nullability

Not knowing have to handle nullability in Room caused my app to crash.

I had the following [Query](https://developer.android.com/reference/androidx/room/Query) which fetched all `OneRMEntities` with a certain `id`.
Since all `oneRMids` were unique this would return at most one `OneRMEntity`.

```
@Query("SELECT * FROM oneRMEntity WHERE oneRMid = :id")
fun getOneRM(id: Long): Flow<OneRMEntity>
```

In my repository I called the method and mapped it to an external model to be
displayed in the UI.

```
override fun getOneRM(id: Long): Flow<OneRMInfo> = oneRMDao.getOneRM(id).map { it asExternalModel() }
```

However, this caused the following crash whenever no `OneRMEntity` with the id was found:

```
java.lang.NullPointerException: Parameter specified as non-null is null: method OneRMEntityKt.asExternalModel, parameter <this>
at OneRMEntityKt.asExternalModel(Unknown Source:2)
```

### The solution

After some research I found the solution. First, I changed the return type to
`OneRMEntity?` to correctly show that it can be null.

```
@Query("SELECT * FROM oneRMEntity WHERE oneRMid = :id")
fun getOneRM(id: Long): Flow<OneRMEntity?>
```

Then I added `filterNotNull()` before mapping it to my external model.

```
override fun getOneRM(id: Long): Flow<OneRMInfo> = oneRMDao.getOneRM(id).filterNotNull().map { it.asExternalModel() }
```

Voila, no more crashes!

## 3. Combining data from two tables

Sometimes it's necessary to combine data from two tables in Room.

In my case I needed information from both `MovementEntity` and `OneRMEntity`, and
created the following `@Query`:

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
fun getMovements(): Flow<Map<MovementEntity, List<OneRMEntity>>>
```

I used [LEFT JOIN](https://www.w3schools.com/sql/sql_join_left.asp) to get all
`MovementEntities` and only the matching `OneRMEntities`.

The method I created to map to my UI model `Movement` looks like this:

```
fun Map.Entry<MovementEntity, List<OneRMEntity>>.asExternalMovement() = Movement(
    id = key.id,
    name = key.name,
    weight = value.maxByOrNull { it.weightInKilos }?.weightInKilos
)
```

The `Movement` gets the id and name from the `MovementEntity` and the
weight from the list of `OneRMEntities`.

There are other ways to combine data from two tables but this has worked well for me so far.

## 4. Using Flow for automatic updates

We often want the UI to always reflect the data in Room, meaning that the UI
should update whenever the data in Room changes. Initially, I solved this without [Flow](https://developer.android.com/kotlin/flow) in the following way.


### Without Flow
In `MovementDao` I had the method `getMovements()` which returned `Map<MovementEntity, List<OneRMEntity>>`.

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
suspend fun getMovements(): Map<MovementEntity, List<OneRMEntity>>
```

This was called from the `MovementRepository` and the result was mapped to a list of `Movement`.

```
override suspend fun getMovements(): List<Movement> =
    movementDao.getMovements().entries.map {
        it.asExternalMovement()
    }
```

In the ViewModel the repository method was called and `_uiState` was updated.

```
fun getMovements() {
    viewModelScope.launch {
        val movements = Success(movementsRepository.getMovements())
        _uiState.update { movements }
    }
}
```

So far so good, but now comes the problem.

### The Problem
If the data in the database would updated either by adding, editing or deleting a `Movement`, then to keep the UI up to date with the state of the database, *`getMovements()` had to be called*.

```
fun addMovement(movement: Movement) {
    viewModelScope.launch {
        val movementId = movementsRepository.setMovement(movement)
	    ...
        getMovements() // Needed to make the UI reflects the database
    }
}

fun editMovement(movement: Movement) {
    viewModelScope.launch {
        movementsRepository.setMovement(movement)
        getMovements() // Needed to make the UI reflects the database
    }
}

fun deleteMovement(id: Long) {
    viewModelScope.launch {
        movementsRepository.deleteMovement(id)
        getMovements() // Needed to make the UI reflects the database
    }
}
```

We can do better! [Flow](https://medium.com/androiddevelopers/room-flow-273acffe5b57) can easily be used to accomplish the same thing in a much better way. Let's make the necessary changes.

### Let it Flow

In the Dao the suspend keyword is removed and the return type is changed to
`Flow<Map<MovementEntity, List<OneRMEntity>>>`.

```
@Query("SELECT * FROM movementEntity LEFT JOIN oneRMEntity ON movementEntity.id = oneRMEntity.movementId")
fun getMovements(): Flow<Map<MovementEntity, List<OneRMEntity>>>
```

In the repository the return type is changed.

```
override suspend fun getMovements(): Flow<List<Movement>> =
    movementDao.getMovements().map { map ->
        map.entries.map {
            it.asExternalMovement()
        }
    }
```

And in the ViewModel `.collect` is used to collect the Flow and listen to changes.

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

That's it! Now `_uiState` will be updated automatically whenever the data in the database changes. All the ugly calls to `getMovements()` can be removed.

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

Being able to debug well is crucial to be able to develop well. Android Studio
has [great support](https://developer.android.com/studio/inspect/database) for
debugging Room. In the App Inspection window it's possible to inspect, query, and modify the
app's databases while the app is running. This is extremely useful and greatly
speeds up development. To get started, simply open the App Inspection window at `View > Tool Windows > App Inspection`, select the Database Inspector tab, and select the running app process from the menu. 

Here is an example of App Inspection of the database of One Rep Max Tracker.
![screenshot]({{asset_path}}/app-inspection.png){:width="800px"}

## Conclusion

Room is an extremely useful tool for Android development. In this article I've
presented five basic things that I learnt while using Room for my app
[One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker). I hope you found it useful! 

<!-- What are some things about Room that you'd like to share? Let me know in the comments!-->
