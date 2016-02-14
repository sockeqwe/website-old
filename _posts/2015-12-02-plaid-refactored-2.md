---
layout: post
published: true
title: Refactoring Plaid App - A reactive MVP Approach (Part 2)
mathjax: false
featured: false
comments: true
headline: Refactoring Plaid App - A reactive MVP Approach (Part 2)
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

This is the second part of how we could refactor the [Plaid](https://github.com/nickbutcher/plaid) app open sourced by Nick Butcher. In this part we are going to enhance the MVP architecture described in the [first part](http://hannesdorfmann.com/android/plaid-refactored-1/) to become truly reactive.

**Preface:** I started the refactoring with the strong belief that I can refactor the whole app. Surprisingly (ironic), it turns out that I was a very naive man. I simply have underestimated the complexity of the app and the number of features. Therefore, I have started writing this blog post even if not everything described here is implemented (refactored). I just want to give some ideas how this could be implemented. My refactored code can be found on [github](https://github.com/sockeqwe/plaid) as well.

# Recap from first part
This is the second part of a blog series about refactoring plaid app. In the [first part](http://hannesdorfmann.com/android/plaid-refactored-1/) we have applied Model-View-Presenter and have introduced an `ItemsLoader` for loading items from different backend endpoints by invoking `RouteCallers`. Please read the [first part](http://hannesdorfmann.com/android/plaid-refactored-1/) for more details.

People also aksed me if there is a need for yet another MVP blog post as there are already many of them availabale. Well, my intention is not write about MVP. Rather I want to discuss how to build a "truly reactive" app (with MVP on top). Let's continue where we left off.

# Source - Filter
In the app's home screen you can open a drawer on the right side of the screen (or by clicking on the filter icon in the Toolbar) to enable/disable different backend endpoints we load items from. We call a backend endpoint `Source`. So we have sources like "Dribbble Popular", "Dribbble Recent", "Designer News Popular" and so on.

![Recap](/images/plaid/source-filters.png)

Please raise your hand if you have used `SharedPreferences` to save bunch of data and not only "user preferences". I'm definitely guilty of having uses SharedPreferences for that kind of stuff, so did Nick Butcher for storing `Sources`. The problem is that the user can add (we will talk about this in a minute), update (enable / disable) and remove `Sources`. Sounds that a database suits better, right?

## Database
Let's refactor that and replace `SharedPreferences` with a `SQLiteDatabase`. Databases are hard. So we could either write pure sql statements or choose the right library to deal with databases. There are many libraries out there, some are better than the others. Since we use RxJava we certainly want to use a library with first class RxJava support. In my opinion ORM are not the right way to go. Why? Because usually ORMs have to be designed to work well on the average use case. But almost every app has a special use case. For example writing an efficient query which resolves relations to other objects (SQL joins?) is already hard enough in native SQL. But how do you do that when using a certain ORM library? You have to spend a lot of time to learn how the ORM library works to write efficient queries. Whenever you change ORM library you have to learn the new ORM library again and again. Therefore, I prefer to work without ORM libraries and to write raw SQL queries and the only thing I have to know is SQL.

Hence, we will use [SQLBrite](https://github.com/square/sqlbrite) from Square which is a lightweight wrapper around `SQLiteOpenHelper` with support for RxJava. We don't want to deal with `Cursors` and `ContentValues`. Therefore, we use [SQLBrite-DAO](https://github.com/sockeqwe/sqlbrite-dao) which provides some abstractions like DAO (Data Access Object) and an annotation processor based tiny object mapper (Cursor to object, not ORM) and `ContentValues` generator. If you are looking for a more high level solution with similar functionality you should have a look at [StorIO](https://github.com/pushtorefresh/storio), but I'm more a low level guy so we will use `SQLBrite + SQLBrite-DAO`.

We define a class `Source` like this:

```java
@ObjectMappable
class Source() { // Unfortunately data class are not supported yet by sqlbrite-dao

    object ID { // ID's for predefined Sources
        const val UNKNOWN_ID = -1L
        const val DESIGNER_NEWS_POPULAR = -2L
        const val DESIGNER_NEWS_RECENT = -3L
        const val DRIBBBLE_POPULAR = -4L
        const val DRIBBBLE_FOLLOWING = -5L
        const val DRIBBLE_MY_SHOTS = -6L
        const val DRIBBLE_MY_LIKES = -7L
        const val DRIBBLE_RECENT = -8L
        const val DRIBBLE_DEBUTS = -9L
        const val DRIBBLE_ANIMATED = -10L
        const val DRIBBLE_MATERIAL = -11L
        const val PRODUCT_HUNT = -12L
    }

    @Column(SourceDao.COL.ID)
    var id: Long = ID.UNKNOWN_ID

    @Column(SourceDao.COL.ORDER)
    var order: Int = 0

    @Column(SourceDao.COL.ENABLED)
    var enabled: Boolean = false

    @Column(SourceDao.COL.AUTH_REQUIRED)
    var authenticationRequired = false

    @Column(SourceDao.COL.BACKEND_ID)
    var backendId: Int = -1

    @Column(SourceDao.COL.NAME)
    var name: String? = null

    @Column(SourceDao.COL.NAME_RES)
    var nameRes: Int = -1
  }
```

`@ObjectMappable` and `@Column` are annotations from the SQLBrite-DAO's annotation processor. Next we define a DAO to manipulate and query `Source` from database like this:

```java
class SourceDao : Dao() {

    object COL {
        const val ID = "id"
        const val ORDER = "orderPosition"
        const val ENABLED = "enabled"
        const val AUTH_REQUIRED = "authRequired"
        const val BACKEND_ID = "backendId"
        const val NAME = "name"
        const val NAME_RES = "nameRes"
    }

    private val TABLE = "Source"

    override fun createTable(database: SQLiteDatabase) {
        CREATE_TABLE(
                TABLE,
                "${COL.ID} INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL",
                "${COL.ENABLED} BOOLEAN",
                "${COL.AUTH_REQUIRED} BOOLEAN",
                "${COL.ORDER} INTEGER",
                "${COL.BACKEND_ID} INTEGER NOT NULL",
                "${COL.NAME} TEXT",
                "${COL.NAME_RES} INTEGER")
                .execute(database)
    }

    fun getAllSources(): Observable<List<Source>> {
        return defer {
            query(
                    SELECT(COL.ID, COL.ENABLED, COL.ORDER, COL.BACKEND_ID, COL.NAME, COL.NAME_RES)
                      .FROM(TABLE)
                      .ORDER_BY(COL.ORDER)
            ).run()
            .mapToList(SourceMapper.MAPPER) // annotation processor generates that
        }
    }

    fun getSourcesForBackend(backendId: Int): Observable<List<Source>> {
        return defer {
            query(
                    SELECT(COL.ID, COL.ENABLED, COL.ORDER, COL.BACKEND_ID, COL.NAME, COL.NAME_RES)
                      .FROM(TABLE)
                      .WHERE("${COL.BACKEND_ID} = ?")
                 ).args(backendId.toString())
                 .run()
                 .mapToList(SourceMapper.MAPPER)
        }
    }

    fun insert(source: Source): Observable<Long> {

      val builder = SourceMapper.contentValues().
                      .id(source.id)
                      .enabled(source.enabled)
                      .name(source.name)
                      .nameRes(source.nameRes)
                      .order(source.order)
                      .authenticationRequired(source.authenticationRequired)
                      .backendId(source.backendId)
                      .build()

        return defer { insert(TABLE, cv)
    }

    fun enableSource(sourceId: Long, enabled: Boolean): Observable<Int> {

        val cv = SourceMapper.contentValues()
                .enabled(enabled)
                .build()

        return defer { update(TABLE, cv, "${COL.ID} = ?", sourceId.toString()) }
    }
}
```

SQLBrite-DAO provides a simple SQL syntax so auto completion in your IDE works and the annotation processor generates a class `SourceMapper` to deal with `Cursor` and `ContetnValues` and SQLBrite wraps that all into RxJava so we can work with Observable.

## SourceFilterFragment in MVP
In the current implementation the right drawer with list of sources is integrated in `HomeActivity`. We will refactor that and apply MVP here as well. We implement `SourceFilterFragment implmenets SourceFilterView` and `SourceFilterPresenter` that subscribes to `SourceDao` like this:

```java
class SourceFilterPresenter(val sourceDao: SourceDao, val presentationModelMapper: (List<Source>) -> List<SourceFilterPresentationModel>) : RxPresenter<SourceFilterView, List<SourceFilterPresentationModel>>() {

    fun loadSources() {

        view?.showLoading(false)

        subscribe(
                sourceDao.getAllSources().map(presentationModelMapper),
                // onError
                {
                    view?.showError(it, false)
                },
                // onNext
                {
                    view?.setData(it)
                    view?.showContent()
                }
        )
    }

    fun changeEnabled(source: SourceFilterPresentationModel) {

        val observable = sourceDao.enableSource(source.sourceId, !source.enabled)

        subscribe(observable,
                { // On Error
                    view?.showError(it, true)
                }) // OnNext not needed

    }
}
```

The original intention of MVP was that the Presenter transforms the Model into a `PresentationModel` which will be displayed in the View. We don't display `List<Source>` in `SourceFilerView` but rather `SourceFilterPresentationModel`. This presentation model is optimized for the view. Do we always need a PresentationModel in MVP? It depends. If your Model is just a POJO than it should be fine to display the Model directly in the View, but please don't "leak" application layer models into the view. Soon or later you will call a method of the application layer model from the View. By having a PresentationModel the view has absolutely no knowledge of the application layer models. However, we use a `SourceFilterPresentationModel` because we have the problem that our `Source` is simply not ready to be displayed in a RecyclerView. The Source class can either have a `int nameRes` (which is `R.string.something`) in case that it is a predefined Source or a `String name` if the Source has been added by the user of the app (more about that later). Furthermore, Source class only contains a `backendId` but we want to display a cell with an icon and title like this:

![SourceFilterItem](/images/plaid/source-filter-item.png)

To display such an item we have to "map" the `backendId` to an icon and for the title "map" `nameRes` to a String or use `name` as the title. Of course you can do that in RecyclerView's adapter in `onBindViewHolder()` but then the complexity of the adapter increases. Furthermore, if you do that in adapters onBindViewHolder() method this "mapping" will be done on androids main ui thread and will be executed during scrolling. That might not be an issue in our use case, but if you have to do some more complex things like sorting elements or compute some properties to be displayed, then you can run into trouble. So we pass a function `presentationModelMapper: (List<Source>) -> List<SourceFilterPresentationModel>` as constructor parameter of `SourceFilterPresenter` and then use RxJava's `map()` operator to map `List<Source>` to `List<SourceFilterPresentationModel>`. Another nice thing about RxJava is it's threading model. Actually, we are doing this mapping async on the background thread that is querying the database and then give the view the resulting PresentationModel on the main ui thread. This mapping function looks like this:

```java
class BackendManager {

    object ID {
        const val DRIBBBLE = 0
        const val DESIGNER_NEWS = 1
        const val PRODUCT_HUNT = 2
    }

    val getBackendIconRes = fun(backendId: Int) = when (backendId) {
        ID.DRIBBBLE -> R.drawable.ic_dribbble
        ID.DESIGNER_NEWS -> R.drawable.ic_designer_news
        ID.PRODUCT_HUNT -> R.drawable.ic_product_hunt
        else -> throw IllegalArgumentException("Unknown Backend for id = ${backendId}")
    }
}

// We pass BackendManager.getBackendIconRes() as backendToIconMap function
class SourceToPresentationModelMapper(private val context: Context, private val backendToIconMap: (Int) -> Int) : (List<Source>) -> List<SourceFilterPresentationModel> {

    override fun invoke(sources: List<Source>): List<SourceFilterPresentationModel> {

        val presentationModels = ArrayList<SourceFilterPresentationModel>()
        sources.forEach { source ->

            val name =
                    if (source.name == null) {
                        context.resources.getString(source.nameRes)
                    } else {
                        source.name!!
                    }

            presentationModels.add(SourceFilterPresentationModel(source.id, backendToIconMap(source.backendId), name, source.enabled))
        }
        return presentationModels
    }
}
```

This is yet another way to define a function (as an class) in kotlin. I just wanted to play around a little bit with kotlin.

# Let's get reactive
Many developers are excited about Rx programming, because Rx programming offers functional alike operators like `flatMap()` etc. Many of them don't understand that by using RxJava they are observing data. Therefore, they don't understand Rx programming at all. It is not only about transforming data. Rx implements the observer pattern. You are subscribing to an observable to get updates. That may be a one time consumable data stream like a http response, but the real power of the observer pattern and RxJava can be seen and used by making a data source observable that can emit items more than once.

## Observable Database
We haven't talked much about SQLBrite yet, but the biggest advantage of SQLBrite is that whenever we change a database table's dataset (insert, update or delete row) we get notified through SQLBrite and RxJava. `SourceFilterPresenter.loadSources()` subscribes to `sourceDao.getAllSources()` which returns an `Observable<List<Source>>`. But that is not just a "single onetime" observable to run the query and complete once the query result has been emitted. No, this Observable will remain subscribed (until unsubscribed) and SQLBrite will recognize table changes and simply rerun the sql query and calls `onNext()` with the new query result set.

We also haven't discussed yet how `SourceFilterPresenter` actually works when subscribing to the `SourceDao`:

  1. When `SourceFilterFragment` starts the `SourceFilterPresenter` get's instantiated and `SourceFilterPresenter.loadSources()` gets called. This calls `SourceDao.getAllSources()` which basically runs `SELECT * FROM Source` and returns an Observable. The Presenter subscribes himself on this observable and `onNext()` gets called with the sql query result. The presenter then tells the view to displays the Sources (PresentationModel) i.e. in a RecyclerView.
  2. When the user clicks on a "source item" in the RecyclerView to enable or disable this source then the view calls `presenter.changeEnabled()`. This will basically do an update on the database like `UPDATE Source SET enabled = ? WHERE id = ?`
  3. SQLBrite will recognize that the dataset of table "Source" has been updated (step 2). Therefore, SQLBrite checks if there are Observable with Subscribers on the same Table and detect that the Observable from step 1 (`SELECT * FROM Source`) is still "alive". Hence, SQLBrite will rerun this sql query and emit the new query result (with the updated source). The presenter receives the new result in subscribers `onNext()` and then updates the view.

An important thing to note is that by clicking in the RecyclerView on an item to enable / disable a Source, the Source object itself in the adapter will not be updated. As discussed above, the query will rerun and emit an entirely new `List<Source>` which then will be mapped to an entirely new `List<SourceFilterPresentationModel>` which then will be displayed in the RecyclerView (replaces the previous List<SourceFilterPresentationModel>). This is by design! By doing so this is the first step to ensure to have immutable objects.

Immutability prevents that someone else is touching your object. You can think of it as watching a baseball game in a stadium. You are sitting somewhere in the middle of the tribune. As the game goes on you get hungry. Luckily there is a vendor nearby selling hot dogs, but he stands some seats away from you. Nevertheless, you order a hot dog. The vendor will give your hot dog to the first person in the row you are sitting and he will pass it to his neighbor, that person will pass your hot dog to his next neighbor and so on until you finally get your hot dog. I'm pretty sure you don't want that someone else has eaten a piece of your hot dog during the way from vendor to you. You want an immutable hot dog! Same is valid for our data objects. We don't want that other components (especially other threads) can change our data objects. You might have already noticed that this means that our `Source` class shouldn't have setters, but actually has. This is my fault in combination with `SQLBrite-DAO` which not fully supports kotlin yet and requires a public setter method for his annotation processor. If you work on a real app I recommend to use Google's [auto-value](https://github.com/google/auto/tree/master/value) to create immutable objects.

## Adding a Source
If you open the search you can "save" this search (click on the fab). That means we create a "Source" with the search string as name (hence a Source can have either nameRes=R.string.foo or name ="Hello"):

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/PPwNV45L3T0" frameborder="0" allowfullscreen></iframe>
</p>

What actually happens under the hood with SQLBrite + RxJava is pretty the same as described before ( enabling / disabling a Source):

![Recap](/images/plaid/inserting-source.png)

## Reactive Routing
As already said: this is the second part of a blog series about refactoring plaid app. In the [first part](http://hannesdorfmann.com/android/plaid-refactored-1/) we have applied Model-View-Presenter and have introduced an `ItemsLoader` for loading items from different backend endpoints by invoking `RouteCallers`. Please read the [first part](http://hannesdorfmann.com/android/plaid-refactored-1/) if you haven't yet for more details. Summarizing, the home screen is build like this:

![Recap](/images/plaid/part1-recap.png)

Please note that we have established an observable unidirectional bottom-up dataflow by using RxJava. We already have discussed that we store `Sources` in a database and that the user can enable and disable sources dynamically. Whenever the user enables / disables a Source in `SourceFilterView` we have to reload the items in the `HomeView`, right? We have seen that the `HomePresenter` is responsible to load data items by using an `ItemsLoader`. So how do we notify the HomePresenter that a Source has been enabled / disabled? Using an EventBus in such a scenario is a common solution.

But there is a better way, a truly reactive way: We don't have to tell the `HomePresenter` about changes at all. How? Well, the HomePresenter is already observing `ItemsLoader`. So the `HomePresenter` doesn't really care about "source changes". All the HomePresenter is interested in is receiving items from his `onNext()` subscriber-callback and display them via `HomeView`. So when a `Source` is changed new items to display will be emitted. Easy (in theory), right? But what does it actually takes to build something like that? Good news: We already have almost everything we need. The `RouteCallerFatory.getAllBackendCallers()` already returns an `Observable<List<RouteCaller>>` (please note that this is an `Observable`). The `Router` passes this Observable to the ItemsLoader and the HomePresenter subscribes on it. So as already said, to bring new items to the HomePresenter's `onNext()` callback we have to emit new items. Which kind of items? New `List<RouteCaller>` because each RouteCaller will be executed by `ItemsLoader` (Page) to load Items from backend endpoints and finally emit the loaded items to `HomePresenter`. In other words: To make the Routing "reactive" we have to make our `RouteCallerFatory` "reactive" by observing the database (SQLBrite) and emitting the updated `RouteCaller` when a Source has been enabled or disabled:

```java
class HomeDribbbleCallerFactory(private val backend: DribbbleService, sourceDao: SourceDao) : RouteCallerFactory<List<PlaidItem>> {

    private val backendCalls = ArrayMap<Long, RouteCaller<List<PlaidItem>>>()
    private val sources: Observable<List<Source>>

    init {
        // Observable for the Database
        sources = defer {
            sourceDao.getSourcesForBackend(BackendManager.ID.DRIBBBLE).share()
        }
    }

    private fun createCaller(source: Source): RouteCaller<List<PlaidItem>> {
        return RouteCaller(0, ITEMS_PER_PAGE, getBackendMethodToInvoke(source))
    }

    // Find the method to invoke (retrofit backend endpoint call) for a given Source
    private fun getBackendMethodToInvoke(source: Source):
            (pageOffset: Int, itemsPerPage: Int) -> Observable<List<PlaidItem>> = when (source.id) {
        // Predefined Sources
        Source.ID.DRIBBBLE_POPULAR -> getPopular
        Source.ID.DRIBBBLE_FOLLOWING -> getFollowing
        Source.ID.DRIBBLE_ANIMATED -> getAnimated
        Source.ID.DRIBBLE_DEBUTS -> getDebuts
        Source.ID.DRIBBLE_RECENT -> getRecent
        Source.ID.DRIBBLE_MY_LIKES -> getMyLikes
        Source.ID.DRIBBLE_MY_SHOTS -> getUserShots

        // Custom Source created by user by searching.
        else -> SearchFunc(source.name!!)
    }

    override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
        return sources.map(mapSourcesToBackendCalls)
    }

    // Map Sources from Database to corresponding RouteCaller
    val mapSourcesToBackendCalls = fun(sources: List<Source>): List<RouteCaller<List<PlaidItem>>> {
        val calls = ArrayList<RouteCaller<List<PlaidItem>>>()
        sources.forEach { source ->

            val call = backendCalls[source.id]

            if (call == null) {
                // New source added
                if (source.enabled) {
                    val newCall = createCaller(source)
                    backendCalls.put(source.id, newCall)
                    calls.add(newCall)
                }

            } else {
                // Already existing source

                if (!source.enabled) {
                    // Source has been disabled
                    backendCalls.remove(source.id)
                } else {
                    calls.add(call)
                }
            }
        }

        return calls
    }


    //
    // Backend endpoint calls
    //

    val getPopular = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getPopular(pageOffset, itemsPerPage)
    }

    val getFollowing = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getFollowing(pageOffset, itemsPerPage)
    }

    val getAnimated = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getAnimated(pageOffset, itemsPerPage)
    }

    val getDebuts = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getDebuts(pageOffset, itemsPerPage)
    }

    val getRecent = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getRecent(pageOffset, itemsPerPage)
    }

    val getMyLikes = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getUserLikes(pageOffset, itemsPerPage)
    }

    val getUserShots = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getUserShots(pageOffset, itemsPerPage)
    }

    private class SearchFunc(val queryString: String) : (Int, Int) -> Observable<List<PlaidItem>> {

        override fun invoke(pageOffset: Int, pageLimit: Int): Observable<List<PlaidItem>> {

            return Observable.defer<List<PlaidItem>> {
                try {
                    // Dribbble API doesn't provide a search endpoint.
                    // Therefore, parse HTML search result manually
                    Observable.just(DribbbleSearch.search(queryString, DribbbleSearch.SORT_RECENT, pageOffset))
                } catch(e: Exception) {
                    Observable.error(e)
                }
            }
        }
    }

}
```

As you see, our `HomeDribbbleCallerFactory` is observing our database (`sourceDao.getSourcesForBackend()`) and whenever the database has been changed because the user has enabled/disabled a Source or have added a new one (custom search) `sources.map(mapSourcesToBackendCalls)` will be called again and emits a new `List<RouteCaller>` with the updated routes to call. Next the `ItemsLoader` (via `Router` and `FirstPage`) will reacting on the new emitted `List<RouteCaller>` and load new Items which finally triggers `HomePresenter's onNext()` callback with the new loaded items. It sounds more complicated than it actually is. Have a look at the following graphically representation:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/pmWjwrLVDdA" frameborder="0" allowfullscreen></iframe>
</p>

The yellow circle represents the data flow. You see that SQLBrite will rerun all queries and inform the subscribers. Since `HomeDribbbleCallerFactory` is an subscriber to the database this component gets notified when a `Source` has been enabled/dispable and automatically adjust the Router. Thanks to RxJava and SQLBrite we can build a **truly reactive application** where the app reacts on changes and at the end updates the UI "magically" by still having an unidirectional data flow (unlikely using an EventBus).


## Posting items to DesignerNews
If you haven't noticed yet: You can submit news to DesignerNews site directly from Plaid app (Floating-Action-Button on home screen). Nick Butcher uses an `IntentService` to execute the http call and a `LocalBroadcastReceiver` (kind of `EventBus`) to notify the `HomeActivity` whether the story has been submitted successfully or not:

![Recap](/images/plaid/posting-old.png)

Using an android `Service` is definitely the way to go to ensure that the story will be posted independent from activities lifecycle. However, there is a small issue with the current implementation: If the user submits a post `PostStoryService` gets started, but this Service will only inform `HomeActivity` after having uploaded successfully and then HomeActivity displays the uploaded item in the RecyclerView with the other uploaded. The problem is that if you have a slow internet connection (i.e. executing the http call to submit the post to DesignerNews takes 10 seconds) the user has no clue or visual feedback whats going on. A good idea would be to display the item immediately in HomeActivity's RecyclerView with a ProgressBar in the items layout (ViewHolder) and simply hide that ProgressBar once the post has been submitted successfully (http call successful). In case of error it would be nice if the failed item in HomeActivity's RecyclerView will be highlighted with an error icon and a retry button. Even better would be to add offline support (no network connection). Sounds very complex, doesn't it? How do you implement that? Using an EventBus?

Don't worry, we can implement that in a "truly reactive" way without additional work to integrate it into our already refactored code. Let's do it step by step. First we have to implement offline support, so we have to save the story we will post on Designer News somehow locally on our device: We use a SQLite database for that and SQLBrite + SQLBrite-DAO again. Let's define a simple data class `NewDesignerNewsStory` that represents a story that we will post on Designer News afterwards:

```java
@ObjectMappable
class NewDesignerNewsStory : PlaidItem {

    object State { // ID's for predefined Sources
        const val NOT_SUBMITTED = 0
        const val IN_PROGRESS = 1
        const val FAILED = 2
    }

    @Column(StoryDao.COL.ID)
    var id : Long

    @Column(StoryDao.COL.TITLE)
    var title: String

    @Column(StoryDao.COL.COMMENT)
    var comment: String

    @Column(StoryDao.COL.URL)
    var url: String

    @Column(StoryDao.COL.STATE)
    var state = State.NOT_SUBMITTED
  }
```

Furthermore, we implement `StoryDao` which is responsible to insert a `NewDesignerNewsStory`, update the state of a `NewDesignerNewsStory`. I'm not going to show the code for that because I guess you know how this SQL statements will look like. Next we will refactor `PostStoryService` to use `StoryDao` to query the local database for not submitted Posts and post them:

{% highlight java %}
class PostStoryService : IntentService ("PostStoryService") {

   @Inject lateinit var storyDao : StoryDao
   @Inject lateinit var backend : DesignerNewsService

   fun onHandleIntent(i : Intent) {
      val toSubmit  = storyDao.getAllNotInProgress() // List<StoryDao> of NOT_SUBMITTED and FAILED
      toSubmit.forEach(submit)
   }

   private val submit = fun(story : NewDesignerNewsStory){
     try {
       storyDao.setUploadInProgress(story.id) // Mark Story as in progress
       backend.postNewStory(story)  // Blocking retrofit call
       storyDao.remove(story.id) // successfully so remove this from database
     } catch (e : IOException){
       storyDao.setFailed(story.id)
     }
   }
}
{% endhighlight %}

Nick Butcher has already implemented a `PostNewDesignerNewsStoryActivity` to have a UI to submit a story. We will refactor this as well and apply MVP as already done with other components before. So at the end there will be a `PostNewDesignerNewsStoryActivity` (View), a `PostNewDesignerNewsStoryPresenter` that will be save  a `NewDesignerNewsStory` into the local database by using `StoryDao`. The idea is to store every new story the user of the plaid app wants to submit into database rather then executing the http call directly and then afterwards start `PostStoryService` to execute the http call. As you see in the code snipped above we will update the state (IN_PROGRESS / FAILED) while trying to submit the story and save that persistently into database. In a nutshell:

![Recap](/images/plaid/offline-support.png)

All right, but you might ask yourself now: How the hack do we display that items from database in our HomeActivity? Well, it's easier than you might have thought. We already have implemented a very flexible routing mechanism which we are already using in `HomeActivity`. So we will simply add another route to our Router: But rather then making http calls to a backend we add a route to our local (offline) database. All it takes is to define a `RouteCallerFactory`:

{% highlight java %}
class OfflineStoryCallerFactory (private val storyDao : StoryDao) : RouteCallerFactory<List<PlaidItem>>{

  private val callers = arrayListOf(RouteCaller(0, 100, callFunc))

  private val callFunc = fun (pageOffset : Int, limit : Int) : Observable<List<PlaidItem>> {
    return storyDao.getAll()
  }

  override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
    return callers
  }

}
{% endhighlight %}

Easy right? Let's add the `OfflineStoryCallerFactory` to the `Router` used for the home screen. In [part 1](http://hannesdorfmann.com/android/plaid-refactored-1/) we have already discussed that we can do that configuration in the corresponding dagger module:

{% highlight java %}
@Module(
    injects = {
        HomePresenter.class
    },
    addsTo = ApplicationModule.class // contains Retrofit interfaces
)
public class HomeModule {

  @Provides @Singleton HomePresenter provideSearchPresenter(SourceDao sourceDao,
      StoryDao storyDao,
      DribbbleService dribbbleBackend,
      DesignerNewsService designerNewsBackend,
      ProductHuntService productHuntBackend ) {

    // Create the router
    List<RouteCallerFactory<List<? extends PlaidItem>>> routeCallerFactories = new ArrayList<>(3);
    routeCallerFactories.add(new HomeDribbbleCallerFactory(dribbbleBackend, sourceDao));
    routeCallerFactories.add(new HomeDesingerNewsCallerFactory(designerNewsBackend, sourceDao));
    routeCallerFactories.add(new HomeProductHuntCallerFactory(productHuntBackend, sourceDao));

    // Add the new route to local database
    routeCallerFactories.add(new OfflineStoryCallerFactory(storyDao));

    Router<List<? extends PlaidItem>> router = new Router<>(routeCallerFactories);

    ItemsLoader<List<? extends PlaidItem>> itemsLoader = new ItemsLoader<>(router);

    return new HomePresenterImpl(itemsLoader);
  }
}
{% endhighlight %}

So at the end we have a `HomePresenter` that gets items from an `ItemsLoader` which `Router` enables him to load and displays items from `Dribbble` (HomeDribbbleCallerFactory), `Designer News` (HomeDesingerNewsCallerFactory), `Product Hunt` (HomeProductHuntCallerFactory) and from the users local (offline) database via `OfflineStoryCallerFactory`. Since `PostStoryService`, `PostNewDesignerNewsStoryPresenter ` and `OfflineStoryCallerFactory` use `StoryDao` which is build on top of SQLBrite + RxJava the whole thing is reactive. That means, whenever we add a new Story to our local database by using `StoryDao`, the `HomePresenter` will automatically be notified as well. It's the same principle as described before when enabling / disabling sources.

![Recap](/images/plaid/posting-new.png)

Note also that `PostStoryService` is changing the the state of a `NewDesignerNewsStory` which will result in updating the UI of `HomeView` (show / hide a ProgressBar on the item that will be uploaded). We also get error handling for free: i.e. when an `NewDesignerNewsStory` couldn't be submitted to Designer News backend we can show a retry button on this item displayed in HomeActivity's RecyclerView. By clicking on the retry button we can simply start `PostStoryService` again which will try to upload that item again. Please note also that we still have an unidirectional data flow.
We can improve that even more by starting `PostStoryService` with an exponential back off in case of bad network connection or use `JobSchedulers` or `GcmNetworkManager` to start `PostStoryService` only when it makes sense (save battery).

# Conclusion of Part 2
In this part we have shown that RxJava (or Rx programming in general) is more than just concatenating Retrofit http calls and transforming data. The observer pattern is nothing new, but that is what makes RxJava so beautiful. Establishing an unidirectional data flow is the key for having clean and easy to debug and maintain architecture. That might be the biggest difference between using RxJava base Observable (once again: OBSERVER PATTERN) and an EventBus, which could do the job as well but you might not have an unidirectional data flow. We could also apply the reactive pattern to authentication which I haven't covered at all.

Please note that the source code is far away from being perfect and I'm pretty sure you will find bugs. So please don't copy & paste blindly this code into your app. I simply hadn't time to cover all cases and refactor all things. I had to publish that blog post to stop myself to spend more days and weeks on the refactoring of this app. You can see the ideas and code posted in this blog as some kind of "prove of concept" of a "truly reactive" app. Once again I want to point out the importance of immutable objects and [pure functions](https://en.wikipedia.org/wiki/Pure_function). Unluckily my source code is not as immutable as it should be. As already said, the focus was set on writing a "reactive" app.

Nevertheless, I'm willing to clean up my code and to make a [pull request](https://github.com/nickbutcher/plaid/issues/49) to Nick Butcher's original repository. However, I don't expect that this "Reactive MVP approach" will be merged into the Plaid apps source code. I do understand that my approach is for advanced developers. Furthermore, I use some third party libraries and kotlin, which may not everybody has knowledge of. Therefore my code is not ideal for all kind of developers, which without any doubt Nick Butcher as a Google employee wants to reach with his awesome app. It's good and important to have such an open source app with an inspiring UI / UX understandable for both android beginners and experts.

I also wanna catch the opportunity to talk about an idea I have in mind for almost two years: It would be very nice to have an app (Plaid alike) with different sources of android development bundled into one app. For example the app could display some tweets and Google+ posts from android dev super stars like Jesse Wilson, Jake Wharton, Matthias Käppler, Piwai, Dan Lew, Corey Latislaw, Cyril Mottier, Lisa Wray, Artem Zinnatullin and others (forgive me if I have forget to mention your name here) but also some other sources like Reddit [/r/androiddev](https://www.reddit.com/r/androiddev/), Stackoverflow, YouTube channels, podcasts etc. bundled into one nice app with an outstanding UI / UX as Plaid. I don't think that this is something a single developer can develop in his spare time (and the experience from refactoring the plaid app gives me right). But if we could find a handful developers interessted in such an app developed by the community for the community I'm definetly willing to contribute as well.

In the next (third and last) part of this series of blog posts about refactoring the Plaid app I will share my experience with kotlin during the refactoring and we are going to discuss how to test such an app, because not only [#perfmatters](https://twitter.com/hashtag/perfmatters?src=hash) but also [#qualitymatters](http://artemzin.com/blog/android-development-culture-the-document-qualitymatters/)
