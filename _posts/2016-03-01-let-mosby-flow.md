---
layout: post
published: true
title: Let Mosby Flow - An alternative to Fragments
mathjax: false
featured: false
comments: true
headline: Let Mosby Flow - An alternative to Fragments
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

The usage of Fragments in Android apps is highly controversial. While some developers love them, others hate them. In this blog post I will give you a short introduction of how to use Mosby 3.0 (SNAPSHOT) to build MVP base screens and square's Flow library as navigation stack replacement.

**Preface:** Usually I use Fragments in my apps and 99% of the time they work well. However, I do understand developers who are advocating against Fragments. We want build the best apps we are able to and if Fragments are a source for errors then this 1% is probably to much.

In this blog post I will show you how to write a little **atlas app** entirely without Fragments by using [Mosby](https://github.com/sockeqwe/mosby) and [Flow](https://github.com/square/flow). Our app has just two screens: A list of countries and a details screen where information about a certain country is displayed. Let's have a look at a short demo video (please note that this demo app is able to deal with screen orientation changes):
<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/lNG1odHSNXg" frameborder="0" allowfullscreen></iframe>
</p>

# Flow for navigation
The app itself will be a single Activity. As already said, there are basically two screens the user can navigate to:

- `CountriesListLayout`: Displays a list of countries. The user can click on a country to display details about this country.
- `CountryDetailsLayout`: Displays details about a certain country like population, currency and some photos.

## Dispatcher and Keys
To integrate flow in your activity you have to do the following (kotlin programming language):

```java
class MainActivity : AppCompatActivity() {

  override fun attachBaseContext(baseContext: Context) {
    val flowContextWrapper = Flow.configure(baseContext, this)
        .dispatcher(AtlasAppDispatcher(this))
        .defaultKey(CountriesScreen())
        .keyParceler(AtlasAppKeyParceler())
        .install()
    super.attachBaseContext(flowContextWrapper)
  }

  override fun onBackPressed() {
    if (!Flow.get(this).goBack()) {
      super.onBackPressed();
    }
  }

  override fun onCreate(savedInstanceState: Bundle?) {
      super.onCreate(savedInstanceState)
      setContentView(R.layout.activity_main)
  }
}
```

Let's start with an easy thing to spot: We override `onBackPressed()` to forward the press of android's back button to Flow.

Just for completeness, `R.layout.activity_main` is just a FrameLayout "container":

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    />
```

Next, we will focus on how to configure Flow. To install Flow in our Activity we have to override `attachBaseContext()`. Why? Well, internally Flow will create a [ContextWrapper](https://developer.android.com/reference/android/content/ContextWrapper.html) and we have to use that special context wrapper returned by Flow for our Activity by calling `super.attachBaseContext(flowContextWrapper)`.

Flow is highly customizable which on one hand is great and allows you to be very flexible. On the other hand that means that you have to write some "boilerplate" code. To tell Flow how to navigate in our app we have to define a `Dispatcher`. The Dispatcher is responsible to "dispatch" changes on Flow's navigation (history) stack.

```java
class AtlasAppDispatcher(private val activity: Activity) : Dispatcher {

  override fun dispatch(traversal: Traversal, callback: TraversalCallback) {
    val destination = traversal.destination.top<Any>() // destination key

    val layoutRes = when (destination) {
      is CountriesScreen -> R.layout.screen_countries
      is CountryDetailsScreen -> R.layout.screen_countrydetails
      else -> throw IllegalStateException("Unknown screen $destination")
    }

    val newView = LayoutInflater.from(traversal.createContext(destination, activity))
        .inflate(layoutRes, container, false)

    // Update container: remove oldView, insert newView
    val container = activity.findViewById(R.id.container) as ViewGroup

    // Remove current screen from container
    if (traversal.origin != null && container.childCount > 0) {
      val currentScreen = container.getChildAt(0);
      // Save the state manually
      traversal.getState((traversal.origin as History).top()).save(currentScreen)
      container.removeAllViews() // remove oldView
    }

    // Restore state before adding view (i.e. caused by onBackPressed)
    traversal.getState(traversal.destination.top()).restore(newView)

    // add new screen
    container.addView(newView)

    callback.onTraversalCompleted() // Tell Flow that we are done
  }
}
```

Alright, let's discuss some key aspects of the code shown above:
As you see we have to implement Flow's interface `Dispatcher` with the method `dispatch()`. This method will be invoked whenever we use Flow to navigate through the app and we have to specify (manually) how to apply the navigation changes to your view. In the Atlas app we have a "container" (FrameLayout) and whenever we navigate from screen to the next screen (or back to previous screen) we simply remove the current screen from the container and add the new screen.
Flow gives us a `Traversal` object as parameter which contains all the information we need to apply navigation stack changes. We get a "key" `destination = traversal.destination.top()` from Flow. Every "screen" is identified by a key and you have to make a mapping from key to a android.view.View like we do here:

```java
val layoutRes = when (destination) {
  is CountriesScreen -> R.layout.screen_countries
  is CountryDetailsScreen -> R.layout.screen_countrydetails
  else -> throw IllegalStateException("Unknown screen $destination")
}
```
You may ask yourself "what is a key"? Basically everything (java.lang.Object) can be used as a key for a screen in Flow. It seems to be good practice that "keys" are named with "Screen" suffix. In our atlas app we have two "screens" we can navigate to. Hence we have two "key" classes named `CountriesScreen` and `CountryDetailsScreen`. This "key" classes have two responsibilities: First, as already discussed a key maps to an android view, and second the key contains all the required data the screen needs to display. I think that can be compared to fragment arguments. For example the `CountryDetailsScreen` contains an id (country id) which then is used in the corresponding view to load the data for the given country id.

```java
class CountryDetailsScreen(val countryId: Int) : Parcelable {

  private constructor(p: Parcel) : this(p.readInt())

  override fun writeToParcel(parcel: Parcel, p1: Int) {
    parcel.writeInt(countryId)
  }

  override fun describeContents(): Int = 0

  companion object {
    val CREATOR = object : Parcelable.Creator<CountryDetailsScreen> {
      override fun createFromParcel(p: Parcel): CountryDetailsScreen = CountryDetailsScreen(p)

      override fun newArray(size: Int): Array<out CountryDetailsScreen>? = Array(size,
          { CountryDetailsScreen(-1) })
    }
  }
}
```

I guess I know your next question: "Why do we need to implement the parcelable interface?". I have said earlier that "everything" can be a key and that's still true. However, at some point (during process death, i.e. activity gets destroyed while in background) Flow has to save your keys persistently in a Bundle (as Parcelable) to be able to restore the navigation stack history after your activity gets restarted (i.e. activity comes in the foreground again). Therefore, we have to provide a `KeyParceler` to Flow which is responsible to write and read a "key" as parcelable. The easiest way to do so is to make the "key" like `CountryDetailsScreen` itself parcelable because then our `KeyParceler` implementation is basically just casting the "key" object like we do in our atlas app:

```java
class AtlasAppKeyParceler : KeyParceler {
  override fun toParcelable(key: Any?): Parcelable = key as Parcelable
  override fun toKey(parcelable: Parcelable) : Any = parcelable
}
```

<small>Please note that the type `Any` is kotlins equivalent to java.lang.Object </small>

To sum it up, to use Flow in our Activity we have to do:

```java
class MainActivity : AppCompatActivity() {

  override fun attachBaseContext(baseContext: Context) {
    val newBase = Flow.configure(baseContext, this)
        .dispatcher(AtlasAppDispatcher(this))
        .defaultKey(CountriesScreen())
        .keyParceler(AtlasAppKeyParceler())
        .install()
    super.attachBaseContext(newBase)
  }
  ...
}
```

With `.defaultKey(CountriesScreen())` we specify what is our start key / screen:

```java
class CountriesScreen : Parcelable // Doesn't have any data, it's just an empty object
```

# MVP with Mosby
Alright, so far we have talked about how to setup Flow as navigation stack replacement.
We haven't discussed yet how to build the UI without Fragments. We simply write custom Views by extending from ViewGroups like FrameLayout etc. Furthermore, we want to have a separation of concerns (separation from UI and business logic). This is the point where Mosby comes in.

Mosby is a Model-View-Presenter (MVP) library for Android. For our atlas app we will use Mosby 3.0, which is not released at the time of writing this blog post. However, 3.0.0-SNAPSHOT is available and changes until the final 3.0 release will mainly be "under the hood". In other words, the API is mostly stable (and compatible to Mosby 2.0).

## Screen orientation changes
One of the most loved features of Mosby is that Presenters can survive screen orientation changes. Additionally, Mosby has a tiny companion object to the Presenter and View called `ViewState`. Typically, in MVP (passive view) the Presenter coordinates the View. So the Presenter tells the View to display a ProgressBar like `view.showLoading()` while loading data and then the RecyclerView `view.showContent()` once the data has been loaded. Mosby's ViewState is some kind of hook sitting between Presenter and View and keeps track of all the methods the presenter has invoked on the view. The idea is that after a screen orientation change we can "apply" the ViewState and invoke the same methods on the View to get back to the UI state as before the screen orientation change.

If you have used Mosby 2.0 before this is nothing new to you. This feature was already available for Activities and Fragments. With Mosby 3.0 this feature is now fully supported for subclasses of android.view.ViewGroup like FrameLayout, RelativeLayout and so on (there was already partial support for that in Mosby 2.0).

Let's have a look how we have implemented the screen that displays a list of countries:


```java
class CountriesListLayout(c: Context, atts: AttributeSet) : CountriesView, MvpViewStateFrameLayout<CountriesView, CountriesPresenter>(
    c, atts) {

  private val recyclerView: RecyclerView by bindView(R.id.recyclerView)
  private val swipeRefreshLayout: SwipeRefreshLayout by bindView(R.id.swipeRefreshLayout)
  private val errorView: View by bindView(R.id.errorView)
  private val loadingView: View by bindView(R.id.loadingView)

  private val adapter = CountriesAdapter(
      { // OnClickListener, navigates to details screen
        country ->
        Flow.get(this).set(CountryDetailsScreen(country.id))
      })

  init {
    // inflates the layout containing a SwipeRefreshLayout, RecyclerView, ProgressBar etc.
    LayoutInflater.from(context).inflate(R.layout.recycler_swiperefresh_view, this, true)

    recyclerView.adapter = adapter
    recyclerView.layoutManager = LinearLayoutManager(context)

    errorView.setOnClickListener {
      loadData(false)
    }

    swipeRefreshLayout.setOnRefreshListener {
      loadData(true)
    }
  }

  fun loadData(pullToRefresh: Boolean) = presenter.loadCountries(pullToRefresh)

  override fun createPresenter(): CountriesPresenter =
    AtlasApplication.getComponent(context).countriesPresenter() // We use dagger 2

  override fun createViewState(): ViewState<CountriesView> = RetainingLceViewState<List<Country>, CountriesView>()

  override fun showLoading(pullToRefresh: Boolean) {
      loadingView.visibility = VISIBLE
      errorView.visibility = GONE
      swipeRefreshLayout.visibility = GONE
  }

  override fun showContent() {
    loadingView.visibility = GONE
    errorView.visibility = GONE
    swipeRefreshLayout.visibility = VISIBILE
    swipeRefreshLayout.isRefreshing = false
  }

  override fun showError(e: Throwable?, pullToRefresh: Boolean) {
    swipeRefreshLayout.visibility = GONE
    loadingView.visibility = GONE
    errorView.visibility = VISIBLE
    swipeRefreshLayout.isRefreshing = false
  }

  override fun setData(data: List<Country>) {
    adapter.items = data
    adapter.notifyDataSetChanged()
  }
}
```

For more details about Mosby you should read [Mosby's documentation](http://hannesdorfmann.com/mosby/). As you might have already noticed we extend from `MvpViewStateFrameLayout` which is provided by Mosby. Since Mosby follows the delegation principle is quite easy to make every ViewGroup work Mosby. All you have to do is to implement `ViewGroupViewStateDelegateCallback` in your custom ViewGroup class and forward "lifecycle events" like `onAttachedToWindow()` and `onDetachedFromWindow()` to Mosby's `ViewGroupMvpDelegate`. This sounds more complex than it actually is. Let's have a look at `MvpViewStateFrameLayouts` source code:

```java
public abstract class MvpViewStateFrameLayout<V, P>
    extends FrameLayout implements ViewGroupViewStateDelegateCallback<V, P> {

  private ViewGroupMvpDelegate mvpDelegate = new ViewGroupMvpViewStateDelegateImpl<V, P>(this);

  @Override protected void onAttachedToWindow() {
      super.onAttachedToWindow();
      mvpDelegate.onAttachedToWindow();
  }

  @Override protected void onDetachedFromWindow() {
      super.onDetachedFromWindow();
      mvpDelegate.onDetachedFromWindow();
  }

  // Implement in subclass
  abstract ViewState<V> createViewState();
  abstract P createPresenter();

}
```

The `CountriesPresenter` loads a List<Country> from Atlas (business logic, injected by dagger 2) and we use RxJava to connect the dots:

```java
class CountriesPresenter @Inject constructor(val atlas: Atlas) : MvpBasePresenter<CountriesView>() {

  var subscription: Subscription? = null

  fun loadCountries(pullToRefresh: Boolean) {
    view?.showLoading(pullToRefresh)
    subscription = atlas.getCountries()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(
            { view?.setData(it) },
            { view?.showError(it, pullToRefresh) },
            { view?.showContent() }
        )
  }

  override fun detachView(retainInstance: Boolean) {
    super.detachView(retainInstance)
    if (!retainInstance) {
      subscription?.unsubscribe()
    }
  }
}
```

As already said, in Mosby the Presenter survives screen orientation changes and the view gets simply attached and detached from the presenter. Mosby is smart enough to detect when the user has been navigated to another screen so that the Presenter will be destroyed permanently.
The screen displaying a country details is basically the same and therefore not worth posting the code here again. You can checkout the whole sample code on [github](https://github.com/sockeqwe/mosby/tree/master/sample-flow).

# Summary
The aim of this blog post was to demonstrate that we can build an app without Fragments by using Flow for navigation and Mosby for MVP. Both, Flow and Mosby can deal with process deaths (Activity destroyed in background), however, Mosby requires to make the ViewState parcelable (that means that the loaded data, i.e. list of country, has to implement parcelable as well, see [documentation](hannesdorfmann.com/mosby/)). I, personally, think that 95% of app developers just want that their app survive screen orientation changes painlessly and therefore a "simple view state" (not implementing parcelable) is enough (if process death occurs then data will be reloaded entirely).

Depending on your app, Flow may requires you to write a lot of code (especially for `Dispatcher`). Nevertheless, Flow is really powerful (still in 1.0-alpha) and we haven't discussed all features of Flow in detail like complex dispatchers with views on top of each other like dialogs or cases where you don't have a single "container" to display a view but rather something similar as child-fragments (Fragment's in Fragments) with back button support  or Flow services or how does Flow save the instance state (you have to save and restore that manually, see `AtlasAppDispatcher`). Also "keys" have to override `equals()` and `hashCode()` properly. In a nutshell: Flow is not designed for android dev beginners, but the benefit of Flow is huge (if you hate fragments)!

If you are looking for something more simple then Flow you might find  [Pancakes](https://github.com/mattlogan/Pancakes) interesting which is also a navigation stack library but not as powerful as Flow. With `Pancakes` you would provide a `ViewFactory` for each "screen" like this:


```java
class CountriesListLayoutFactory implements ViewFactory {
    @Override
    public View createView(Context context, ViewGroup container) {
        return LayoutInflater.from(context).inflate(R.layout.screen_countries, container, false);
    }
}
```

Mosby should work with Pancakes  (and any other navigation stack library) as great as with Flow.

One last thing: people asked me why Mosby doesn't provide it's own navigation stack implementation? The reason is that Mosby should and will ever be that tiny little MVP library and that's what Mosby is the best at. Take Mosby as a base scaffold to build your app on top of it with that development stack you like the most (like Flow or Pancakes for navigation or even with Fragments). Furthermore, implementing a clean navigation stack library is not that easy as it seems or why do you think that the brilliant guys over at square took quite a long time to design and implement Flow?


<small>Please note that for better readability the code shown in this blog post has been cut down a little bit. Check the demo repository for the complete code</small>
