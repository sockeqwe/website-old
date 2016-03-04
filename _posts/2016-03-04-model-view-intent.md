---
layout: post
published: true
title: Model-View-Intent on Android
mathjax: false
featured: false
comments: true
headline: Model-View-Intent on Android
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

As developers we should always think outside the box. Some weeks ago Artem Zinnatullin and I have discussed some architectural trends on android and on other platforms, like .NET and javascript, in his Podcast [The Context](https://github.com/artem-zinnatullin/TheContext-Podcast). A few days later [Christina Lee](https://twitter.com/RunChristinaRun) gave an awesome lightning talk [Redux-ing UI Bugs](https://www.youtube.com/watch?v=UsuzhTlccRk) during square's **The Journey of Android Engineers** event. She also talked about one of my favorite js libraries: [Cycle.js](http://cycle.js.org/), which defines itself as **Model-View-Intent (MVI)** library. After watching Christina Lee's inspiring talk I finally found motivation to take the time to write down my thoughts about MVI on android.

**Preface:** Model-View-Intent relies heavily on reactive and functional programming (RxJava). Believe me, many people won't understand MVI at the first time. I was in the same situation. A year ago, I definitely gave up and started again several times to understand it, but guess what: that's completely fine! If you don't understand this blog post at the first go, it's ok, relax, take it easy and retry it a few days later again.

So what is Model-View-Intent? Yet another architecture for user interfaces? Well, I would recommend you to watch the following video about cycle.js presented by the inventor [André Staltz](https://twitter.com/andrestaltz) himself at JSConf Budapest in May 2015 (at least the first 10 minutes) to get a better understanding of the motivation behind MVI:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/1zj7M1LnJV4" frameborder="0" allowfullscreen></iframe>
</p>

I know it's javascript but the key fundamentals are the same for all UI based platforms, incl. android (just replace DOM with android layouts, UI widgets, ...).

# From MVC to MVI

MVI is inspired by some other js frameworks like redux and react, but the key principle comes from Model-View-Controller (MVC). I mean the original MVC introduced 1979 by Trygve Reenskaug to separate View from Model. Again, to clarify, I'm not talking about MVC from iOS (ViewController) or Android (Activity / Fragment) or backend frameworks which also (mis)use the word controller. I talk about MVC in his pure original form. A Model that is observed by the View and a Controller that manipulates the Model. Unlikely nowadays, in Reenskaug original idea of MVC almost every UI element has it's own Model and Controller. Imagine a CheckBox UI widget: this one observers his own model, i.e. a boolean. The controller could be a OnCheckboxChangedListener triggered by the user's mouse. From cycle.js docs:

> The Controller in MVC is incompatible with our reactive ideals, because it is a proactive component (implying either passive Model or passive View). However, the original idea in MVC was a method for translating information between two worlds: that of the computer’s digital realm and the user’s mental model.

![Model-View-Controller](/images/mvi/mvc.png)

<small>Source: cycle.js.org</small>

However, in the original MVC the controller may or may not also manipulate the view. But that is not exactly what we want, because we want an **unidirectional data flow** and **immutability** to establish predictable states, which leads to cleaner, more maintainable code and less bugs.

![Circle](/images/mvi/mvi-cicle.png)

<small>Source: cycle.js.org</small>

Do you see the unidirectional flow? the cycle? The next question is how do we establish such a circle? Well, as you have seen above, the computer takes an input and converts it to an output (display / view). The human, sees the output from computer and takes it as Input and produces Output (UI widgets events like a click on a button) which then will be again the input for the computer. So the concept of taking a input and have an output seems to be familiar, doesn't it? Yes it's a (mathematically) function. We can establish that with **functional programming**.

So what we basically want to have is a chain of functions like this:

![MVI](/images/mvi/mvi-func1.png)


- `intent()`: This function takes the input from the user (i.e. UI events, like click listener) and translate it to "something" that will be passed as parameter to `model()` function. This could be a simple string to set a value of the model to or more complex data structure like an `Actions` or `Commands`. Here in this blog post we will stick with the word `Action`.
- `model()`: The model function takes the output from `intent()` as input to manipulate the model. The output of this function is a new model (state changed). So it should not update an already existing model. **We want immutability!** We don't change an already existing one. We copy the existing one and change the state (and afterwards it can not be changed anymore). This function is the only peace of your code that is allowed to change a Model object. Then this new immutable Model is the output of this function.
- `view()`: This method takes the model returned from `model()` function and gives it as input to the  `view()` function. Then the view simply displays this model somehow.

But what about the cycle, one might ask? This is where reactive programming (RxJava, observer pattern) comes in.

![MVI](/images/mvi/mvi-func2.png)

So the view will generate "events" (observer pattern) that are passed to the `intent()` function again.

Sounds quite complex, I know, but once you are into it it's not that hard anymore. Let's say we want to build a simple android app to search github (rest api) for repositories matching a certain name. Something like this:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/ZXwDQML5IXQ" frameborder="0" allowfullscreen></iframe>
</p>

Let's have a look at a very naive implementation (kotlin). We will use Jake Whartons [RxBinding library](https://github.com/JakeWharton/RxBinding) to get RxJava Observables from `SearchView` widget. Our data model class looks like this:

```java
data class SearchModel(val searchTerm: String, val results: List<GithubRepo>)
```

And our main Actvitiy:

```java
class SearchActivity : AppCompatActivity() {

  val githubBackend : GitHubBackend = ... ; // Retrofit for Github Rest API
  val editSearch: android.widget.SearchView by bindView(R.id.searchView)
  val recyclerView: RecyclerView  by bindView(R.id.recyclerView)
  val loadingView: View by bindView(R.id.loadingView)
  var adapter: SearchResultAdapter

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_search)
    setSupportActionBar(toolbar)

    adapter = SearchResultAdapter(layoutInflater)
    recyclerView.adapter = adapter
    recyclerView.layoutManager = LinearLayoutManager(this)

    // MVI: please note that the following code is chained togehter
    // the comments and empty lines are just for legibility

    // "intent()" maps UI event to "action" (a search string)
    Observable<String> = RxSearchView.queryTextChangeEvents(editSearch)
           .filter { it.queryText().count() >= 3 }
           .debounce(500, TimeUnit.MILLISECONDS)  // here: Observable<String>

    // "model()" loads data from model
          .startWith("")  // If starting the first time we emit empty string as Model
          .flatMap { queryString ->
            if (queryString.isEmpty())
              Observable.just( SearchModel("", emptyList)) // returns Observable<SearchModel>
            else
              githubBackend.search(queryString) // Retrofit; here: Observable<GithubResponse>
                            .map {
                                response ->
                                SearchModel(queryString, response.items)
                            }   // here: Observable<SearchModel>
          } // end flatMap; here: Observable<SearchModel>

    // "view()" method is just subscribing and updating UI
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ result -> // SearchModel ; onNext()
      adapter.items = result.items
      adapter.notifyDataSetChanged()
      },
      { error -> // onError()
        Toast.makeText(...);
      }
    )

  }
}
```

That's a lot of code isn't it. I hope you could follow that code and find my comments helpful.
In a nutshell: `intent()` function basically listens to SearchView Text changes and gives the query string (kind of action to tell the model to search for that string) to the `model()` function.
The `model()` function is responsible to manage the "model". With `startWith()` we ensure that the first time we subscribe to an "empty" search result gets forwarded (in other words, we setup the initial state). Otherwise we use retrofit to load a `GithubResponse` that we than have to transform to a `SearchModel`. Last the `view()` gets the SearchModel as result (RxJava Observer) and is responsible to "render" and display the SearchModel.

Great, we have a unidirectional data flow. Even better, we have immutability and pure functions (none of this function has side effects or stores or changes the state somehow, except retrofit).
So are we done?


# The Big Picture
What about all the other things we have learned from other architectural design patterns like MVP? What about concepts they offer like separation of concerns, decoupled & maintainable code, testability, reusability, single responsibility and so on?

You get it, this code is basically  a common android developer beginner mistake: Put everything in one huge Activity ([God object](https://en.wikipedia.org/wiki/God_object)). Well, it has a structure thanks to MVI and (mainly RxJava), but is it good code? It's not spaghetti code (maybe reacitve spaghetti code, that's a matter of opinion), but is it good code?

Nevertheless, we can do it better. So let's refactor this code.
As you might already know I'm a fan of MVP. So lets combine the best of both, MVP and MVI.
In this sample we will use Mosby, the MVP library. Before we start, let's talk about the separation of concerns a little bit. What is the responsibility of the View in MVP? Right, just display the data, and doing what the Presenter "commands" to display. In MVI what should the GUI do? Exactly, the GUI triggers UI events and the `intent()` function will translate that to "actions" to manipulate the model afterwards.

So the MVP View Interface will offer a `intent()` function, in our case we call this method `searchIntent()` function:

```java
interface SearchView : MvpView {
  fun searchIntent(): Observable<String> // intent() function
}
```
Next our Activity will become a MVP View (implements `SearchView`). That means, everything that is not related to updating the UI or generate UI events will be removed.

```java
class SearchActivity : SearchView, MvpActivity<SearchView, SearchPresenter>() {

  val editSearch: android.widget.SearchView by bindView(R.id.searchView)
  val recyclerView: RecyclerView  by bindView(R.id.recyclerView)
  val loadingView: View by bindView(R.id.loadingView)
  var adapter: SearchResultAdapter

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_search)
    setSupportActionBar(toolbar)

    adapter = SearchResultAdapter(layoutInflater)
    recyclerView.adapter = adapter
    recyclerView.layoutManager = LinearLayoutManager(this)
  }

  // Provide a intent() function
  override fun searchIntent(): Observable<String> {
      return RxSearchView.queryTextChangeEvents(editSearch)
           .filter { it.queryText().count() >= 3 }
           .debounce(500, TimeUnit.MILLISECONDS)
    }

  // dependency injection via dagger
  override fun createPresenter(): SearchPresenter = App.getComponent(this).searchPresenter()
}
```
Ah, looks much better now, doesn't it? Let's continue with the Presenter.
The Presenters responsibility is to coordinate the view and to be the "bridge" to the business logic. In Mosby, a Presenter has two methods, `attachView()` (called from activity.onCreate() ) and `detachView()` (called from activity.onDestroy()).

```java
class SearchPresenter : MvpBasePresenter<SearchView>() {

  lateinit var subscription: Subscription

  override fun attachView(view: SearchView) {
    super.attachView(view)

    subscription = view.getSearchIntent()
          .startWith("")  // model() function, same as before
          .flatMap { queryString ->
            if (queryString.isEmpty())
              Observable.just( SearchModel("", emptyList)) // returns Observable<SearchModel>
            else
              githubBackend.search(queryString) // Retrofit; here: Observable<GithubResponse>
                            .map {
                                response ->
                                SearchModel(queryString, response.items)
                            }   // here: Observable<SearchModel>
          } // end flatMap; here: Observable<SearchModel>
          .observeOn(AndroidSchedulers.mainThread())
          .subscribe(view.showData(), view.showError())
  }

  override fun detachView(retainInstance: Boolean) {
    super.detachView(retainInstance)
    subscription.unsubscribe()
  }
}
```

Alright, so now the Presenter uses the View's `serachIntent()` method and connects it to the model. Are we done? No, now the presenter has the business logic code as well. So there is one separation of concern still missing. We will refactor that in a minute. Let's continue with this little statement first: `.subscribe(view.showData(), view.showError())`. Well, in MVP the Presenter tells the view what to display. So what are this two methods? This methods are part of the `SearchView` interface that I have omitted before:

```java
interface SearchView : MvpView {
  fun searchIntent(): Observable<String> // intent() function
  fun showData(): (SearchModel) -> Unit // RxJava Action1
  fun showError(): (Throwable) -> Unit  // RxJava Action1
}
```

```java
class SearchActivity : SearchView, MvpActivity<SearchView, SearchPresenter>() {

  ...

  override fun showData(): (SearchModel) -> Unit = {
     adapter.items = it.results
     adapter.notifyDataSetChanged()
   }

   override fun showError(): (Throwable) -> Unit = {
     loadingView.visibility = View.GONE
     Toast.makeText(this, "An Error has occurred", Toast.LENGTH_SHORT).show()
     it.printStackTrace()
   }
}
```

So what we now have and what we didn't had before doing the refactoring is a entirely decoupled View. All the View has to provide is an `Observable<String>` as `intent()` function. Today the result comes from a `SearchView` and has a `debounce()` operator. Tomorrow it could be something entirely different (i.e. a dropdown menu with a list of strings) and you don't have to touch (and therefore can't break) anything but `SearchActivity`. The same for the way how the view is "rendered" / displayed. All that code lives in `SearchActivity` as well. So today it is uses a `RecyclerView`, but tomorrow it could be another custom UI widget or a ListView. I guess you get the point.

Back to our Presenter source code: as already said, the `Presenter` has all the business logic. One of the main pitfalls with that is that presenter is not testable (how to mock parts of our busines logic) and we can't reuse that business logic for other Presenter because it's hard coded. Let's refactor that code. First we introduce a `SearchEngine`:

```java
class SearchEngine(private val githubBackend: GithubBackend) {

  fun search(query: String): Observable<SearchModel> =
      if (query.isEmpty()) {
        Observable.just(SearchModel("", emptyList()))
      } else {
        githubBackend.getRepositories(query).map { SearchModel(query, it.items) }
      }
}
```

`SearchEngine` get a `GithubBackend` and offes a `search(String) : Observable<SearchModel>` for the ourside. This basically is our `model()` function, responsible to change the model. However, we don't want to hardcode that again in our presenter. So we use Dagger to provide and inject a `modelFunc()` to other components, in our case to the `searchPresenter`:

```java
@Module
class ApplicationModule {

  @Provides @Singleton
  fun providesSearchEngine(): SearchEngine {
    val retrofit = Retrofit.Builder()
          .baseUrl("https://api.github.com")
          .addConverterFactory(MoshiConverterFactory.create())
          .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
          .build()
    return SearchEngine(retrofit.create(GithubBackend::class.java))
  }

  @Provides @Singleton
  fun providesModelFunc(
      searchEngine: SearchEngine): Function1<Observable<String>, Observable<SearchModel>> =
      { stringObservable ->
        stringObservable.startWith("").flatMap { queryString -> searchEngine.search(queryString) }
      }
}
```

The important bit here is `providesModelFunc()` which offers a Lambda `Observable<String> -> Observable<SearchModel>`. Since lambdas are just anonymous functions (kind of) we take this lambda and inject it into `SearchPresenter`:

```java
class SearchPresenter @Inject constructor(
                      val modelFunc: (Observable<String>) -> Observable<SearchModel>
                    ) : MvpBasePresenter<SearchView>() {

  lateinit var subscription: Subscription

  override fun attachView(view: SearchView) {
    super.attachView(view)

    subscription =
        modelFunc(  // model()
              view.searchIntent()  // intent()
            )
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe( // view()
              view.showData(),
              view.showError()
          )
  }

  override fun detachView(retainInstance: Boolean) {
    super.detachView(retainInstance)
    subscription.unsubscribe()
  }
}
```

That's it. we still have `view( model( intent() ) )`, but this time the presenter is super slim, decoupled, testable and maintainable.

# The problem with side effects
Are we done now? Almost. We haven't discussed yet who is responsible to display and hide the `ProgressBar` while loading in background data. In MVP it would be the responsibility from `Presenter` to coordinate the view's state ... ah, the View's state ... do you hear the  alarm bells ringing?

Let's see, how could we do that in mvp? We would simply add a method to the MVP View interface like this:

```java
interface SearchView : MvpView {
  ...
  fun showLoading(): (Any) -> Unit  // RxJava Action1
}

class SearchActivity : SearchView, MvpActivity<SearchView, SearchPresenter>() {
  ...
  override fun showData(): (SearchModel) -> Unit = {
     adapter.items = it.results
     adapter.notifyDataSetChanged()

     loadingView.visibility = View.GONE
   }

   override fun showLoading(): (Any) -> Unit = {
     loadingView.visibility = View.VISIBLE
   }
}
```

Then the presenter could do something like this:

```java
class SearchPresenter @Inject constructor(
                      val modelFunc: (Observable<String>) -> Observable<SearchModel>
                    ) : MvpBasePresenter<SearchView>() {

  lateinit var subscription: Subscription

  override fun attachView(view: SearchView) {
    super.attachView(view)

    subscription =
        modelFunc(  // model()
              view.searchIntent()  // intent()
              .doOnNext(view.showLoading()) // Show loading
            )
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe( // view()
              view.showData(),
              view.showError()
          )
  }

  override fun detachView(retainInstance: Boolean) {
    super.detachView(retainInstance)
    subscription.unsubscribe()
  }
}
```

What's wrong with that code? I mean we do that in MVP all the time? The problem is, that now our whole system has two states: The view's state and the state from the model itself. Moreover, the view's state is caused by a side effect. Remember the definition of `model()` function? Only the model function is allowed to change the internal state (with side effects). But the code shown above contradicts with that principle.

So how to solve that? From my point of view there are two options:
The first is to create a MVI flow just for `LoadingView`. We talked about MVC's original definition by Reenskaug. Do you remember? Every GUI widget has it's own Controller. But who says that every controller has to be his own model? We could share the Model. Guess what, sharing an Observable is pretty easy in RxJava, because there is an operator for that called `.share()`. So we could have our own MVI / MVP flow with just the `ProgressBar` as View. In general: **we should stop thinking that the whole screen is one huge view with one controller / presenter and one model**.

The second option, and in my opinion the best option, would be to have one Model that also propagates his stat changes, i.e. before loading data from github the `modelFun()` would change the model to it's internal state "loading":

```java
data class SearchModel(
      val isLoading: Boolean, // true before loading data, false when done
      val searchTerm: String,
      val results: List<GithubRepo>)
```

Now `SearchEngine` who provides the `model()` function would first change the model to `SearchModel (true, ...)` when starting to load the data (and propagate this state change as usual via observable chain which will update the view) and then set it to `SearchModel (false, )` after having retrieved the new data from github backend.

# Screen Orientation Changes

[Sir Tony Hoare](https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions) introduced null references in ALGOL W back in 1965 and called it his _"Billion Dollar Mistake"_ (NullpointerException). I think android's _"Billion Dollar Mistake"_ is to destroy the whole Activity on screen orientation changes. I think process death is fine, but dealing with screen orientation changes that way it is today in android is really painful. Furthermore, it makes software architecture on android much harder then it has to be.

MVI makes no difference here. On screen orientation changes everything will get lost. So that means that our `Model` which is representative for the state of an MVI powered app will be lost. But we could put that in a retaining Fragment somehow. But that only solves half of the problem, because whenever the activity gets destroyed we also have to unsubscribe our observable chain otherwise we will run into memory leaks (we use `view.searchIntent()`). So what if we start a async background task and want to ensure that it doesn't get canceled in our `model()` function? Well we could use `.cache()` operator or `Subjects` like `ReplaySubject` or keep a static map as cache for background tasks. In a nutshell, yes there are ways (I would rather call them workarounds) but I'm not very happy with those solutions. We also have to take into account that our `model()` function may have to distinguish between initial empty state (`.startWith("")`) and state after screen orientation changes and process deaths (do we have to make our model Parcelable to save it persistently in a Bundle?) You see, it's getting out of hand very quickly and introduces more complexity than it has to be.

TL;DR: There might be "workarounds", but I can't recommend you a clean solution of how to deal with screen orientation changes in android.

# Summary
Model-View-Intent is a very clean way how to deal with states and UI changes in your application. Having a unidirectional data flow (cycle) with immutability is awesome. Composing functions leads to clean and reusable code.

The source code for the sample shown in this blog post can be found on [Github](https://github.com/sockeqwe/Model-View-Intent-Android)
