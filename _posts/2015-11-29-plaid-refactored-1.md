---
layout: post
published: true
title: Refactoring Plaid App - A reactive MVP Approach (Part 1)
mathjax: false
featured: false
comments: true
headline: Refactoring Plaid App - A reactive MVP Approach (Part 1)
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

Nick Butcher has open sourced on [github](https://github.com/nickbutcher/plaid) an awesome app called Plaid. The app is pretty cool and has an outstanding UI / UX. Whenever source code of such awesome apps are available developers start to copy code and best practice tips from it. So I did the same and decided to dive into the code of Plaid. As always, you will find some parts of the code that you think could be implemented or be written in another way. So instead of just talking about code I have decided to spend some of my spare time to refactor some parts of Plaid's source code. This blog post is the first part of a series of 3 posts of how I would refactor the Plaid app and to share my thoughts.

**Preface:** I started the refactoring with the strong belief that I can refactor the whole app. Surprisingly (ironic), it turns out that I was a very naive man. I simply have underestimated the complexity of the app and the number of features. I discovered some features only after having spent some hours with Nick Buchter's code, which were not "visible" to me just by using the app. In short: I realized that I don't have the time to put all my thoughts into action. Hence my "refactoring" is focused on the apps "homepage" and the "search screen", which I have refactored mostly. Nevertheless, I want discuss (in theory) some more things that could be refactored, but I didn't because of time constraints. My refactored code can be found on [github](https://github.com/sockeqwe/plaid) as well.

# A fist look
The overall user experience and user interface is pretty awesome. I can't describe it even better than this tweet:
<p>
<blockquote class="twitter-tweet" data-cards="hidden" lang="de"><p lang="en" dir="ltr">If you think Material Design is all about cards and shadows, take a look at Plaid and think again <a href="https://t.co/IYO3QnxkFu">https://t.co/IYO3QnxkFu</a></p>&mdash; Luis G. Valle (@lgvalle) <a href="https://twitter.com/lgvalle/status/661509155455410176">3. November 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
</p>

It's a joy to use the app. The UI / UX is truly an inspiration for every developer and designer.

However, after playing around a little bit more with the app I faced some visual bugs:

- Displaying a loading indicator and error message at the same time:
  <p>
  <iframe width="420" height="315" src="https://www.youtube.com/embed/zCwESjEpNdk" frameborder="0" allowfullscreen></iframe>
  <p/>

- In the app you can apply some "filters" or in other words: You can specify which "sources" of Dribbble, Designer News and Product Hunt you want to display. If you deselect such sources while loading an http request is currently running you can run into the state where the app displays items, but shouldn't since no sources are selected:
<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/nJ3VUjpW0N0" frameborder="0" allowfullscreen></iframe>
</p>

- Also the app doesn't handle screen orientation changes properly. It simply rebuilds the entire screen so that after each orientation change you see the loading indicator again, and re-execute the http calls:
<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/tuIDrtvL0lg" frameborder="0" allowfullscreen></iframe>
</p>

Typically such "issues" are a first indicator for spaghetti code and a moderate software architecture. So let's take a look under the hood and check out the source code and the components for displaying a list of items: The [HomeActivity](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/HomeActivity.java) (about 750 lines of code) handles the visibility of UI elements like displaying the `RecylcerView` or the `ProgressBar`. This activity also decides when to display the Source-Filters-Drawer (on the right side). Furthermore, in it's `onActivityResults()` it does a lot of things inclusive posting new post to designers news. Last but not least it also loads the data for the selected filters by using a [DataManager](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/data/DataManager.java). You see, the `HomeActivity` has many responsibilities, probably too many. The `DataManager` basically uses a combination of `Retrofit` and `AsyncTasks` to execute http calls to load the data. The tricky thing here is pagination. Whenever the end of the `RecyclerView` has been reached, more (older) items are loaded. `DataManager` uses internally a HashMap to track the current page each source (backend endpoint like "Dribbble Popular" or "Dribbble Recent" or "Designer News Popular") is currently displaying. The items are displayed in a RecyclerView by using [FeedAdapter](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/FeedAdapter.java). The [SearchActivity](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/SearchActivity.java) is working quite similar as `HomeActivity`: It uses a `DataManager` and a `FeedAdapter` as well.

# The architecture
From my point of view there is no clear architecture in the current implementation. `HomeActivity` is some kind of god object that manages many things. Another "issue" is that `HomeActivity` changes the UI state by calling the same (internal) methods from different other methods and different events, i.e. the method `checkEmptyState()` get's called from 4 different point's in `HomeActivity` source code.

We will refactor that by applying a Model-View-Presenter with a passive view. The passive view will only be display that things that the presenter tells to do. I'm a fan of MVP with a passive view. People ask me from time to time why I recommend to use passive view and not supervising controller or other MVP derivations. Well, if you use MVP without passive view, you basically are splitting the spaghetti code formerly sitting in the view to half presenter and half view.

## HomeActivity in MVP
So with MVP + passive view we split the responsibilities into two classes: `HomeActivity implements HomeView` is now considered as View (passive view) and implements this interface:

{% highlight kotlin %}
interface HomeView : MvpView {

    fun showLoading()

    fun showContent()

    fun showError()

    fun setContentItems(items: List<PlaidItem>)

    fun showLoadingMore(showing: Boolean)

    fun showLoadingMoreError(t: Throwable)

    fun addOlderItems(items: List<PlaidItem>)

    fun showNoFiltersSelected();
}
{% endhighlight %}

From now on the job of `HomeActivity` is to manage the UI elements (visibility, coordinate animations, etc.) but only after the presenter told to do so. So the state of the View is managed by the `HomePresenter`. The `HomePresenter` looks like this:

{% highlight kotlin %}
class HomePresenter(private val itemsLoader: ItemsLoader<List<PlaidItem>>) : RxPresenter<HomeView, List<PlaidItem>>() {


    fun loadItems() {

        view?.showLoading()

        subscribe(
                itemsLoader.firstPage(),
                { // onError
                    view?.showError()
                },
                { // onNext
                    view?.addOlderItems(it)
                    view?.showContent()
                }
        )

    }

    fun loadMore() {
        view?.showLoadingMore(true)

        subscribe(
                itemsLoader.olderPages(),
                { // onError
                    view?.showLoadingMoreError(it)
                },
                { // onNext
                    view.addOlderItems(it)
                    view.showLoadingMore(false)
                }
        )
    }
}
{% endhighlight %}

If you haven't noticed yet: We use [Kotlin](https://kotlinlang.org/) as programming language, mainly because I like the language and to have the chance to see how development with kotlin works in a "real world" application. Thanks to the kotlin's interoperability I can easily reuse Nick Butcher's java source code, mainly for UI / View things.

For implementing MVP we use [Mosby](http://hannesdorfmann.com/mosby/), a MVP library, which also allows us to keep the presenters during screen orientation changes. So we don't have to restart with loading data and we don't see the `ProgressBar` after screen orientation changes. Mosby allows us to keep the views state as before the orientation change.

Last but not least I have decided to use [RxJava](https://github.com/ReactiveX/RxJava) for my "Model" (hence the `subscribe()` method in the presenter). So the `ItemsLoader` is my refactored and reactive version Nick Butcher's `DataManager`. I will explain the `ItemsLoader` in a minute.

## SearchActivity in MVP
As already said, running a search is very similar to `HomeActivity`. It displays a list (grid) of items and adds pagination to load more items when the end of the `RecyclerView` has been reached. So applying MVP to `SearchActivity` is pretty the same as shown before:

{% highlight java %}
public interface SearchView extends MvpView {

  void showLoading();

  void showContent();

  void showError(Throwable t);

  void setContentItems(List<PlaidItem> items);

  void showLoadingMore(boolean showing);

  void showLoadingMoreError(Throwable t);

  void addOlderItems(List<PlaidItem> items);

  void showSearchNotStarted();
}
{% endhighlight %}

{% highlight kotlin %}
class SearchPresenter(private val itemsLoaderFactory: SearchItemsLoaderFactory) : RxPresenter<SearchView, List<PlaidItem>>() {

    private var itemsLoader: ItemsLoader<List<PlaidItem>>? = null

    fun search(query: String) {

        // Create items loader for the given query string
        itemsLoader = itemsLoaderFactory.create(query)

        view?.showLoading()

        subscribe(itemsLoader!!.firstPage(), { // Error handling
            view?.showError(it)
        }, { // onNext
            view?.setContentItems(it)
        }, {
            view?.showContent()
        })
    }

    fun searchMore(query: String) {

        view?.showLoadingMore(true)
        subscribe(itemsLoader!!.olderPages(), { // Error handling
            view?.showLoadingMore(false)
            view?.showLoadingMoreError(it)
        }, { // onNext
            view?.addOlderItems(it)
        }, { // onComplete
            view?.showLoadingMore(false)
        })

    }

    fun clearSearch() {
        // Unsubscribe any previous search subscriptions
        unsubscribe()

        view.showSearchNotStarted()
    }
}
{% endhighlight %}

The only thing that you might have noticed is that `SearchPresenter` takes a `SearchItemsLoaderFactory` as constructor parameter and creates a `ItemsLoader` dynamically for each search query. We will see how this works in a minute.

# ItemsLoader and Pagination
So far we have covered View and Presenter. Now lets discuss how we could refactor the "Model" or  use case / interactor if you want to compare it with uncle bobs clean architecture.

Before we start: There is a class called `PlaidItem` (holds properties like title and image url). This class is the base class to represent a single item:

- `Shot extends PlaidItem` for an item loaded from Dribbble
- `Story extends PlaidItem` for an item loaded from Designer News
- `Post extends PlaidItem` for an item loaded from Product Hunt

So now let's discuss how we rewrite `DataManager` more efficiently by using RxJava. I don't use RxJava because I think it's cool and all the cool kids have to use RxJava nowadays. You will (hopefully) see the benefits of using RxJava afterwards (especially in the second part of this blog post series).

The hard part of loading items is that we support pagination and loading items from different backends. So let's build the `ItemsLoader` from "bottom-up". At the "bottom" we will find a Retrofit interface for executing the http call. Right now in Plaid you can search `DesignerNews` and `Dribbble`. The pagination problem: Dribbble loads 100 items and requires such a call `loadItems(0, 100)` and the next page will be `loadItems(100, 200)` while DesignerNews increments his page by 1  `loadItems(0, 1)` and next page will be `loadItems(1, 2)`. We need a common API for both. Since we use kotlin we can pass "method references" (function poitners) or lambdas. So what we need is a component that takes such a function, executes this function and returns an `Observable` where we get the result of the actual http call. So basically we need something like this: `backendMethodToCall: (Int, Int) -> Observable<T>`, where the first int parameter is the page offset, the second int parameter the limit (how many items should be loaded per page) and `T` is the generic type of the result (but in fact we are always loading `List<PlaidItem>`).

We call that component `RouteCaller`:

{% highlight kotlin %}
class RouteCaller<T>(private val startPage: Int = 0,
                     private val itemsPerPage: Int,
                     private val backendMethodToCall: (Int, Int) -> Observable<T>) {

    /**
     * Offset for loading more
     */
    private val olderPageOffset = AtomicInteger(startPage)

    /**
     * A queue that is used to retry "older"
     * pages if they have failed before continue with even more older
     */
    private val olderFailedButRetryLater: Queue<Int> = LinkedBlockingQueue<Int>()

    /**
     * Get an observable to load older data from backend.
     */
    fun getOlderWithRetry(): Observable<T> {

        val pageOffset = if (
        olderFailedButRetryLater.isEmpty()) {
            olderPageOffset.addAndGet(itemsPerPage)
        } else {
            olderFailedButRetryLater.poll()
        }

        return backendMethodToCall(pageOffset, itemsPerPage)
                .doOnError {
                    olderFailedButRetryLater.add(pageOffset)
                }
    }

    /**
     * Get an observable to load the newest data from backend.
     * This method should be invoked on pull to refresh
     */
    fun getFirst(): Observable<T> {
        return backendMethodToCall(startPage, itemsPerPage)
    }
}
{% endhighlight %}

`RouteCaller` takes such a method `(Int, Int) -> Observable<T>` as constructor parameter and will call this method internally with the correct parameters:
 - `getFirst()` loads the first page
 - `getOlderWithRetry()`: This method is responsible to load older items. We keep track of the current page in the field `olderPageOffset` and we will increment this one when we start loading more items (or in other words: load an older page). Additionally, we use `.doOnError()` to retry to load the failed page when loading the next page.

So the responsibility of `RouteCaller` is to fill out the parameters (page offset and page limit) needed for the real http backend endpoint call. So what we have is something like this:

![RouteCaller](/images/plaid/routing1.png)

For executing a search we have two backends to query `DesignerNewsService` and `DribbleService`. That means that we have two `RouteCallers` (one for each backend search method to invoke):

![RouteCaller](/images/plaid/routing2.png)

The next question is: How do we instantiate a `RouteCaller`? We define a `RouteCallerFactory` for each backend, which basically offers a method `getAllBackendCallers()` where we get an Observable `List<RouteCaller>` we should execute to load items.

{% highlight kotlin %}
interface RouteCallerFactory<T> {

    /**
     * Get all available backend route callers
     */
    fun getAllBackendCallers(): Observable<List<RouteCaller<T>>>
}
{% endhighlight %}

For executing a search we have `DesignerNewsSearchCallerFactory` and `DribbbleSearchCallerFactory`: The `DesignerNewsSearchCallerFactory` looks like this:

{% highlight kotlin %}
class DesignerNewsSearchCallerFactory(private val searchQuery: String, private val backend: DesignerNewsService) : RouteCallerFactory<List<PlaidItem>> {

    val extractPlaidItemsFromStory = fun(story: StoriesResponse): List<PlaidItem> {
        return story.stories
    }

    // The method to execute from RouteCaller
    val searchCall = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.search(searchQuery, pageOffset).map(extractPlaidItemsFromStory)
    }

    // Create a list with one single RouteCaller() with "searchCall" as method reference
    private val callers = arrayListOf(RouteCaller(0, 100, searchCall))

    override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
        return Observable.just(callers)
    }
}
{% endhighlight %}

At first glance `DesignerNewsSearchCallerFactory` looks a little bit strange if you are new to kotlin because we don't use lambdas but instead we create a property `searchCall` which actually is a function `(Int, Int) ->  Observable<List<PlaidItem>>`.

The reason why we do this is testability: Recently I have watched the fireside chat of Android Dev Summit 2015 where the guys from the Android Team has been asked when they will add Java 8 support. Then Reto Meier, from the Android Team, answered that many developers are mainly interessted in lambdas and asked the audience if they could raise the hand if they want lambdas for android development: Almost the whole audience has raised the hand. I think that there is a general misunderstanding with lambdas: the real power of programming language that offer lambdas are not lambdas itself, is higher order functions and the ability to passing method references. Lambdas are just kind of anonymous functions. Actually, lambdas are not testable, because they are hardcoded. For example if I would have implemented something like this:

{% highlight kotlin %}
RouteCaller(0, 100, { pageOffset, limit -> backend.search(searchQuery, pageOffset) })
{% endhighlight %}

How do we write a unit test for that lambda? It's not possible to write a unit test for that lambda since the lambda is "hardcoded", whereas if we pass a function reference we can easily unit test a function. In java 8 we can pass a method reference by writing `this::searchCall`. Unfortunately, this syntax is not supported [yet](https://youtrack.jetbrains.com/issue/KT-6947#tab=History) in kotlin (right now `::` only supports static methods). Therefore this "workaround" by defining a function as property. More about my experience with kotlin in the last part of this series of blog post.

All right, so for executing a Search we have something like this:

![RouteCallerFactory](/images/plaid/routing3.png)

Please note that for the search `getAllBackendCallers()` returns a list containing just one `RouteCaller`, but the idea is that a `RouteCallerFactory` creates all `RouteCallers` to all available endpoints of a certain backend. As we will see later, the `HomeDribbbleCallerFactory` returns a list of `RouteCallers` for each Dribbble endpoint to load items like popular items, recent, animated etc. So we have something like that:

![RouteCallerFactory](/images/plaid/routing4.png)

Next we introduce a `Router` which is responsible to combine all `RouteCaller` from different `RouteCallerFactories` to one single Observable list:

{% highlight kotlin %}
class Router<T>(private val routeFactories: List<RouteCallerFactory<T>>) {

    fun getAllRoutes(): Observable<List<RouteCaller<T>>> {
        val callers = ArrayList<Observable<List<RouteCaller<T>>>>()

        routeFactories.forEach {
            callers.add(it.getAllBackendCallers())
        }

        return Observable.combineLatest(callers, { calls ->
            val items = ArrayList<RouteCaller<T>>(calls.size)
            calls.forEach {
                items.addAll(it as List<RouteCaller<T>>)
            }

            items // return items
        })
    }
}
{% endhighlight %}

As you see the `Router` takes a list of `RouteCallerFactory`, in other words: a `Router` can be configured via constructor. So to complete the "routing picture":

![Router](/images/plaid/routing5.png)

Ok, so far we have covered the "routing part". So at the end the `Router` offers an `Observable<List<RouteCaller<T>>>`. But when do we finally load items to display them in the `RecyclerView`. This is the responsibility of `ItemsLoader`. As the name already suggests this component loads items:

{% highlight kotlin %}
class ItemsLoader<T>(protected val router: Router<T>) {

    fun firstPage(): Observable<T> {
        return FirstPage<T>(router.getAllRoutes()).asObservable()
    }

    fun olderPages(): Observable<T> {
        return OlderPage<T>(router.getAllRoutes()).asObservable()
    }
}
{% endhighlight %}

`The ItemsLoader` takes a `Router` as constructor parameter and offers two methods:
 - `firstPage()`: Returns an Observable representing the first page.
 - `olderPages()`: Returns an Observable to load older pages.

A `Page` represents a page of items displayed in the RecyclerView. If we scroll to the end of the RecyclerView we load the next page containing older items. Let's have a look at the `FirstPage extends Page` and `OlderPage extends Page` classes:

{% highlight kotlin %}
abstract class Page<T>(val routeCalls: Observable<List<RouteCaller<T>>>) {

    private var failed = AtomicInteger()
    private var backendCallsCount: Int = 0

    /**
     * Return an observable for this page
     */
    @RxLogObservable
    fun asObservable(): Observable<T?> {

        return routeCalls.flatMap { routeCalls ->

            backendCallsCount = routeCalls.size

            val observables = arrayListOf<Observable<T>>()

            routeCalls.forEach { call ->
                    val observable = getRouteCall(call).onErrorResumeNext { error ->
                      // Suppress errors as long as not all fail
                        error.printStackTrace()
                        val fails = failed.incrementAndGet()

                        if (fails == backendCallsCount) {
                            Observable.error(error) // All failed so emmit error
                        } else {
                            Observable.empty() // Not all failed, so ignore this error and emit nothing
                        }
                    }
                    observables.add(observable);
                }

                // return the created Observable
                Observable.merge(observables)
        }
    }

    protected abstract fun getRouteCall(caller: RouteCaller<T>): Observable<T>
}


class FirstPage<T>(routeCalls: Observable<List<RouteCaller<T>>>) : Page<T>(routeCalls) {

    override fun getRouteCall(caller: RouteCaller<T>): Observable<T> {
        return caller.getFirst();
    }
}

class OlderPage<T>(routeCalls: Observable<List<RouteCaller<T>>>) : Page<T>(routeCalls) {

    override fun getRouteCall(caller: RouteCaller<T>): Observable<T> {
        return caller.getOlderWithRetry();
    }
}
{% endhighlight %}

`Page` is responsible for finally executing the http call by invoking `RouteCaller's` method. We also don't want that one entire page fails just because one backend call has fails. Therefore we use `onErrorResumeNext()` to "intercept" errors and only return an error if all http calls have failed.

Then the `Presenter` subscribes to the "page" observable via `ItemsLoader`.

# Dependency Injection
You might have already noticed that almost all components described so far takes other components as constructor parameter. This is by design. Now, we use Dagger (I used Dagger1) to compose such element as needed:

{% highlight java %}
@Module(
    injects = {
        HomePresenter.class
    },
    addsTo = ApplicationModule.class // contains Retrofit interfaces
)
public class HomeModule {

  @Provides @Singleton HomePresenter provideSearchPresenter(SourceDao sourceDao,
      DribbbleService dribbbleBackend,
      DesignerNewsService designerNewsBackend,
      ProductHuntService productHuntBackend ) {

    // Create the router
    List<RouteCallerFactory<List<? extends PlaidItem>>> routeCallerFactories = new ArrayList<>(3);
    routeCallerFactories.add(new HomeDribbbleCallerFactory(dribbbleBackend, sourceDao));
    routeCallerFactories.add(new HomeDesingerNewsCallerFactory(designerNewsBackend, sourceDao));
    routeCallerFactories.add(new HomeProductHuntCallerFactory(productHuntBackend, sourceDao));

    Router<List<? extends PlaidItem>> router = new Router<>(routeCallerFactories);

    ItemsLoader<List<? extends PlaidItem>> itemsLoader = new ItemsLoader<>(router);

    return new HomePresenterImpl(itemsLoader);
  }
}
{% endhighlight %}

As you see, we can use Dagger to configure our `Router` and `ItemsLoader`. For the `SearchPresenter` we configure an `ItemsLoader` and `Router` in `SearchModule`. The advantage is that if we one day  would like to add another "source", like reddit, to display items from reddit, the only thing we have to do is to define a `RedditService` (Retrofit), a `RedditCallerFactoy` and add this CallerFactory to the Router. We can do that in Dagger without having to touch another already existing component's source code ([Open-Closed principle](https://en.wikipedia.org/wiki/Open/closed_principle)). In other words: we have build a "plugin system".

You might have noticed the `SourceDao` class in the code shown above. We will talk about that in the second part of this blog series when we are going "truly reactive".

# Conclusion of Part 1
This is the first part of a series of blog post tree parts. In this first part we have build the fundament by applying Model-View-Presenter and have refactored the way how the app loads data from the backend endpoints. The main idea is to cut this huge and complex task down into several smaller and reusable components like `ItemsLoader`, `Page`, `Router` and `RouteCaller` which follows more the SOLID principle.

As always, there are better ways to implement such an app. Especially, the `ItemsLoader` can be done entirely different. My first intention was to create an unlimited Observable for load older pages by using RxJava's `switchOnNext()` or `merge` operators as described by [Matthias Käppler](https://gist.github.com/mttkay/24881a0ce986f6ec4b4d), but I came to the conclusion that some things regarding UI and error handling are slightly easier to implement if I can split the single observable into two observables (one for first page, one for older page).

As always, feedback and suggestions are very welcome!

In the next (second) part of this series of blog posts about refactoring the Plaid app we will go "truly reactive" by using the real power of RxJava. Stay tuned.
