---
layout: post
published: true
title: In-App Navigation with Coordinators
mathjax: false
featrued: false
comments: true
headline: In-App Navigation with Coordinators
categories:
  - Android
tags: [android, software-architecture, in-app-navigation]
---
Over the last years we have established best practices for writing android apps: clean architecture, architectural patterns like MVP, MVVM, MVI, Repository pattern and so on. But what about in-app navigation? In this blog post I would like to talk about the Coordinator pattern and how we could apply this pattern in android development to organize our in-app navigation code.
 
The [Coordinator pattern](http://khanlou.com/2015/01/the-coordinator/) is a common pattern in iOS development introduced by Soroush Khanlou to help organizing in-app navigation code, 
which seems to be inspired by [Application Controller](https://martinfowler.com/eaaCatalog/applicationController.html) (part of the book [patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html) Martin Fowler et al.).

The goals of this pattern is:
- Avoiding so called Massive ViewControllers (think God-Activity) with way to much responsibilities.
- Give in-app navigation flow logic a home.
- Reuse ViewControllers (think Activity or Fragments) because they are not coupled to in-app navigation.


Before we get deeper into what the Coordinator pattern is and how it can be implemented, let's take a look at what the current state of in-app navigation in android development is.

## Navigation logic in Activity or Fragments
Since the Android SDK requires an Context to start a new Activity (or FragmentManager to put a new Fragment on the back stack) it's quite common to put in-app navigation code directly into your Activity like this (you can find such code in the official android [guides](https://developer.android.com/training/basics/firstapp/starting-activity) provided by Google):

{% highlight java %}
class ShoppingCartActivity : Activity() {  
  override fun onCreate(b : Bundle?){
    super.onCreate(b)
    setContentView(R.layout.activity_shopping_cart)
    val checkoutButton = findViewById(R.id.checkoutButton)
    checkoutButton.setOnClickListener {
      val intent = Intent(this, CheckoutActivity::class.java)
      startActivity(intent)
    }
  }
}
{% endhighlight %}

In-app navigation logic is "hard coded" in **ShoppingCartActivity** which means navigation is tightly coupled to that activity.
Can we test this code easily?
One could argue that we could decouple that by having something like a **Navigator** that can be injected (dependency injection) like this:

{% highlight java %}
class ShoppingCartActivity : Activity() {
  @Inject lateinit var navigator : Navigator  
  override fun onCreate(b : Bundle?){
    super.onCreate(b)
    setContentView(R.layout.activity_shopping_cart)
    val checkoutButton = findViewById(R.id.checkoutButton)
    checkoutButton.setOnClickListener {
      navigator.showCheckout(this)
    }
  }
}
{% endhighlight %}

{% highlight java %}
class Navigator {
   fun showCheckout(activity : Activity){
    val intent = Intent(activity, CheckoutActivity::class.java)
    activity.startActivity(intent)
  }
}
{% endhighlight %}

Okay slightly better but can we do it even better?


## Navigation in MVVM or MVP
Let me ask you a question: if you use MVP or MVVM where do you put in-app navigation logic? 

- The layer below Presenter (let's call it business logic)? Not a good idea because the chances are high that you are going to reuse or share parts of your business logic in other different ViewModels or Presenters.

- In View layer? Do you like playing ping pong between View and Presenter (or ViewModel)?

{% highlight java %}
class ShoppingCartActivity : ShoppingCartView, Activity() {
  @Inject lateinit var navigator : Navigator
  @Inject lateinit var presenter : ShoppingCartPresenter

  override fun onCreate(b : Bundle?){
    super.onCreate(b)
    setContentView(R.layout.activity_shopping_cart)
    val checkoutButton = findViewById(R.id.checkoutButton)
    checkoutButton.setOnClickListener {
      presenter.checkoutClicked()
    }
  }

  override fun navigateToCheckout(){
    navigator.showCheckout(this)
  }
}
{% endhighlight %}

{% highlight java %}
class ShoppingCartPresenter : Presenter<ShoppingCartView> {
  ...
  override fun checkoutClicked(){
    view?.navigateToCheckout(this)
  }
}
{% endhighlight %}

Or if you prefer MVVM over MVP you might use [SingleLiveEvents](https://github.com/googlesamples/android-architecture-components/issues/63) or [EventObserver](https://medium.com/google-developers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)

{% highlight java %}
class ShoppingCartActivity : ShoppingCartView, Activity() {
  @Inject lateinit var navigator : Navigator
  @Inject lateinit var viewModel : ViewModel

  override fun onCreate(b : Bundle?){
    super.onCreate(b)
    setContentView(R.layout.activity_shopping_cart)
    val checkoutButton = findViewById(R.id.checkoutButton)
    checkoutButton.setOnClickListener {
      viewModel.checkoutClicked()
    }

    viewModel.navigateToCheckout.observe(this, Observer {
      navigator.showCheckout(this)
    })
  }
}
{% endhighlight %}

{% highlight java %}
class ShoppingCartViewModel : ViewModel() {
  val navigateToCheckout = MutableLiveData<Event<Unit>>
  fun checkoutClicked(){
    navigateToCheckout.value = Event(Unit) // Trigger the event by setting a new Event as a new value
  }
}
{% endhighlight %}

- So let's put the Navigator into our ViewModel (or Presenter) rather than using EventObserver as shown above?

{% highlight java %}
class ShoppingCartViewModel @Inject constructor(val navigator : Navigator)  : ViewModel() {
  fun checkoutClicked(){
    navigator.showCheckout()
  }
}
{% endhighlight %}

Please note that the code snipped shown above can be translated to Presenter too (injecting Navigator as constructor parameter to Presenter). 
Also, we ignore the fact that Navigator could leak the Activity if it internally holds a reference to it (that is solvable somehow with a workaround).

## Coordinators
So where do we put in-app navigation logic? Business Logic is not a good idea (as described above) and playing ping pong between View and ViewModel (or Presenter) might work but doesn't seem to be a elegant solution. 
Moreover, View still has navigation related responsibilities even if it's just calling **navigator**. 
Therefore, doing navigation in the ViewModel seems to be a considerable alternative but is it really the responsibility of the ViewModel (or Presenter) to care about navigation? 
Shouldn't it be just the glue to present data?
That's why we introduce a Coordinator. 
Yet another level of abstraction? 
Is it really worthwhile? 
Maybe not for a small app but for more complex apps or apps that run A/B tests this could be useful. 
Even if a user can create an account and sign in you have already some major navigation logic -
somewhere you have to check if the user is logged in or not and navigate to the login screen or to the main screen, right?
A Coordinator can be useful in this case.
Also note that Coordinators are not helping you to write less code, they help you to organize in-app navigation related code by giving it a home (and take that responsibility out of View or ViewModel).

The idea is simple: a Coordinator just knows to which screen to go next. 
For example by clicking on the checkout button the Coordinator gets notified and knows where to go next (Checkout). 
It's that simple. 
However, in iOS development it seems to be common to use Coordinator to create ViewControllers, service locator (or dependency injection) and back stack management. 
That's quite a bit for a Coordinator (single responsibility?). 
On Android the operating system instantiates Activities, we have Dagger for dependency injection and we can use Activity or Fragment back stack.
Therefore, I would like to go back to the roots of a Coordinator: A Coordinator just knows where to go next.

## Case Study: A newspaper app using the Coordinator pattern
Finally: let's talk about concrete Coordinators. 
Let's say we have to build a small application for a newspaper with a simple in-app navigation flow:
As a user you see a list of news articles.
Once you click on an article the app opens a new screen where you can read the full article.

![NewsFlow](/images/coordinators/NewsSimpleFlow.png)

{% highlight java %}
class NewsFlowCoordinator (val navigator : Navigator) {

  fun start(){
    navigator.showNewsList()
  } 

  fun readNewsArticle(id : Int){
    navigator.showNewsArticle(id)
  }
}
{% endhighlight %}

A Flow contains one or more screens. 
In our case the "news flow" consists of two screens: news list and read the full news article.
That's it. It's so simple we don't need a library. 
Whenever we start the app, we call **NewsFlowCoordinator.start()** to show the list of all news articles. 
Once the user clicks on a news article **NewsFlowCoordinator.readNewsArticle(id)**  gets called and that news article gets displayed.
We still have a Navigator (we will talk about it in a minute) where we delegate the actual "screen swapping" work to.
A Coordinator is completely stateless, independent from the underlying back stack implementation and only has one responsibility principle: It handles where to go next.

The next question is how do we connect a Coordinator with our ViewModel? 
We follow the push don't pull principle: We pass a lambda (think callback or click listener) into the ViewModel that is triggered once the user clicks on a news article in the UI like that:

{% highlight java %}
class NewsListViewModel(
  newsRepository : NewsRepository, 
  var onNewsItemClicked: ( (Int) -> Unit )?
) : ViewModel() {
  
  val newsArticles = MutableLiveData<List<News>>

  private val disposable = newsRepository.getNewsArticles().subscribe { 
      newsArticles.value = it
  }

  fun newsArticleClicked(id : Int){
    onNewsItemClicked!!(id) // call the lambda
  }

  override fun onCleared() {
    disposable.dispose()
    onNewsItemClicked = null // to avoid memory leaks
  }
}
{% endhighlight %}

**onNewsItemClicked: (Int) -> Unit** is just a lambda that takes an integer as input and returns Unit. 
We pass that lambda as nullable into our ViewModel, hence the surrounding **( ... )?**. This allows us to clear the reference to that lambda to avoid memory leaks.
The next question is: what is actually happening by invoking this lambda? 
It's the navigation flow handler, in other words: a callback to the **NewsFlowCoordinator**.
Whoever creates the **NewsListViewModel** (i.e. Dagger) has to pass in the function **NewsFlowCoordinator::readNewsArticle** like this:

{% highlight java %}
return NewsListViewModel(
  newsRepository = newsRepository,
  onNewsItemClicked = newsFlowCoordinator::readNewsArticle
)
{% endhighlight %}

If you are not an Kotlin expert let me quickly explain you what's going on here. 
Whenever the user clicks on a item in the news list **viewModel.newsArticleClicked(id)** gets called which then invokes **onNewsItemClicked(id)** which is actually the function **readNewsArticle(id)** of NewsFlowCoordinator. 
So a click on a news list item triggers eventually **NewsFlowCoordinator.readNewsArticle(id)**.

The next question is: how is **Navigator** implemented?
This is mostly left as an exercise for the reader because it depends on your concrete use case and personal preferences.
In this example we use a single Activity with multiple Fragments (each screen is a Fragment with corresponding ViewModel). 
Hence a very naive implementation could look like this:

{% highlight java %}
class Navigator{
  var activity : FragmentActivity? = null

  fun showNewsList(){
    activty!!.supportFragmentManager
      .beginTransaction()
      .replace(R.id.fragmentContainer, NewsListFragment())
      .commit()
  }

  fun showNewsDetails(newsId: Int) {
    activty!!.supportFragmentManager
      .beginTransaction()
      .replace(R.id.fragmentContainer, NewsDetailFragment.newInstance(newsId))
      .addToBackStack("NewsDetail")
      .commit()
    }
}
{% endhighlight %}

{% highlight java %}
class MainActivity : AppCompatActivity() {
  @Inject lateinit var navigator : Navigator

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    navigator.activty = this
  }

  override fun onDestroy() {
    super.onDestroy()
    navigator.activty = null // Avoid memory leaks
  }
}
{% endhighlight %}

The presented Navigator implementation is not perfect at all. 
The focus of this blog post is the Coordinator pattern, not the Navigator implementation.
One thing to note though is that since Navigator and NewsFlowCoordinator are stateless they can be in application wide scope (think @Singleton scope in Dagger) and can be instantiated for example in Application.onCreate().

So let's add user authentication functionality to our newspaper app. 
We define a new login screen (LoginFragment + LoginViewModel, we skip "forgot password" and "sign up" in this blog post) and a LoginFlowCoordinator.  
Why not add this functionality to **NewsFlowCoordinator**?
We don't want to have a God-Coordinator, right? 
Also, login related screens belong to a different navigation flow (not reading news), right?

{% highlight java %}
class LoginFlowCoordinator(
  val navigator: Navigator
) {
  fun start(){
    navigator.showLogin()
  }

  fun registerNewUser(){
    navigator.showRegistration()
  }

  fun forgotPassword(){
    navigator.showRecoverPassword()
  }
}
{% endhighlight %}


{% highlight java %}
class LoginViewModel(
  val usermanager: Usermanager,
  var onSignUpClicked: ( () -> Unit )?,
  var onForgotPasswordClicked: ( () -> Unit )?
) {
  fun login(username : String, password : String){
    usermanager.login(username, password)
    ...
  }
  ...
}
{% endhighlight %}

There are two things in **LoginViewModel** worthwhile to discuss: First for every kind of navigation we define it's own lambda (sign up and forgot password) that gets invoked once the user clicks on the corresponding UI widget.
But why is there no "login successful" lambda? 
How do we trigger navigation to next screen after the user is authenticated?  
This is yet another implementation detail (there totally could be a "login successful" lambda) I think it makes more sense to do the following: We add a new **RootFlowCoordinator**  and observe the **Usermanager** (business logic) for changes. 
The Usermanager provides API to sign in and to subscribe (observe, i.e. RxJava) for the current user.

{% highlight java %}
class RootFlowCoordinator(
  val usermanager: Usermanager,
  val loginFlowCoordinator: LoginFlowCoordinator,
  val newsFlowCoordinator: NewsFlowCoordinator,
  val onboardingFlowCoordinator: OnboardingFlowCoordinator
) {

  init {
    usermanager.currentUser.subscribe { user ->
      when (user){
        is NotAuthenticatedUser -> loginFlowCoordinator.start()
        is AuthenticatedUser -> if (user.onBoardingCompleted)
                                    newsFlowCoordinator.start()
                                else
                                    onboardingFlowCoordinator.start()
      }
    }
  }

  fun onboardingCompleted(){
    newsFlowCoordinator.start()
  }
}
{% endhighlight %}

As the name already suggests, **RootFlowCoordinator** becomes our new starting point (not NewsFlowCoordinator.start() anymore).
Let's take a closer look at the RootFlowCoordinator. 
As already mentioned RootFlowCoordinator observes the **Usermanager** whether or not the current user is authenticated.
If user is not authenticated, then we start the **LoginFlowCoordinator**. 
If the user is authenticated (note that if the login in LoginFragment was successful it will propagated to this piece of code, hence no "login successful" lambda in LoginViewModel) then we check if the user already did the onboarding (we will talk about it in a minute) and start the **NewsFlowCoordinator**, otherwise we start the **OnboardingFlowCoordinator**. 
Please note that starting a coordinator doesnt mean creating a new instance, it means calling the start() method.
Let's take a look at the Onboarding flow

![OnboardingFlow](/images/coordinators/OnboardingFlow.png)

{% highlight java %}
class OnboardingFlowCoordinator(
  val navigator: Navigator,
  val onboardingFinished: () -> Unit // this is RootFlowCoordinator.onboardingCompleted()
) {

  fun start(){
    navigator.showOnboardingWelcome()
  }

  fun welcomeShown(){
    navigator.showOnboardingPersonalInterestChooser()
  }

  fun onboardingCompleted(){
    onboardingFinished()
  }
}
{% endhighlight %}

Onboarding starts with  **OnboardingFlowCoordinator.start()** which shows WelcomeFragment (WelcomeViewModel).
Once the use clicks the "next button" **OnboardingFlowCoordinator.welcomeShown()** is invoked which then shows the next screen (PersonalInterestFragment + PersonalInterestViewModel) where the user can set categories of news he is interested in.
Once categories are selected and user clicked "next button" **OnboardingFlowCoordinator.onboardingCompleted()** is invoked which is then actually invoking **RootFlowCoordinator.onboardingCompleted()** which then calls **NewsFlowCoordinator.start()**.
This is how we solve parent-child relations with coordinators: lambdas (or callbacks) to the parent coordinator.

I have mentioned before that Coordinators are useful for A/B tests. 
Let's add a screen that asks the user to do an in app purchase extend our NewsFlowCoordinator so that either the user sees that new screen or not depending on if he is part of group B of the A/B test.

![NewsFlowWithABTest](/images/coordinators/NewsFlowAB.png)

{% highlight java %}
class NewsFlowCoordinator (
  val navigator : Navigator,
  val abTest : AbTest
) {

  fun start(){
    navigator.showNewsList()
  } 

  fun readNewsArticle(id : Int){
    navigator.showNewsArticle(id)
  }

  fun closeNews(){
    if (abTest.isB){
      navigator.showInAppPurchases()
    } else {
      navigator.closeNews()
    }
  }
}
{% endhighlight %}

Again, no navigation logic is in your View or ViewModel but rather the Coordinator knows how to deal with A/B tests.
Do you have to add the **InAppPurchaseFragment** to the onboarding flow too? 
You can do that because the InAppPurchaseFragment nor the corresponding ViewModel is coupled to navigation related code and therefore it is possible to reuse Fragments and ViewModel in other flows.
All we have to pass to the ViewModel is a different lambda as navigation callback.
Is your A/B test bigger than just adding one screen, for example two different onboarding flows you want to A/B test? 
No problem, just create a OnboardingFlowACoordinator and OnboardingFlowBCoordinator. 

You can find the source code [on Github](https://github.com/sockeqwe/CoordinatorsAndroid). If you are too lazy to compile and run the app yourself, here is a little video of the final result:

<p>
<iframe width="560" height="315" src="https://www.youtube.com/embed/PfRLZeRLvTo?ecver=1" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</p>

**Pro Tip:**
With Kotlin you can create nice DSLs for your Coordinators. 
This makes your in-app navigation code even more readable because basically you are creating a navigation graph.

{% highlight java %}
newsFlowCoordinator(navigator, abTest) {

  start {
    navigator.showNewsList()
  } 

  readNewsArticle { id ->
    navigator.showNewsArticle(id)
  }

  closeNews {
    if (abTest.isB){
      navigator.showInAppPurchases()
    } else {
      navigator.closeNews()
    }
  }
}
{% endhighlight %}


## Conclusion
Coordinators can help you to organize in-app navigation logic by creating loosely coupled components with single responsibility and great testability. 
Coordinators can be scoped similar to singleton because they are stateless and you don't create new navigation flows at runtime, therefore you can "hard code" all your in app navigation flows with very readable Kotlin DSL's. 
Are Coordinators on Android ready for prime time? 
As already said, this is not a library, this is just an idea and concept. 
Is this idea applicable in your app?
I dont know, ultimately it's your app and you know best if there is need for the Coordinator pattern and how easy it is to integrate it into your existing app architecture.
Maybe it's good idea is to create a small sample application to try this pattern out.

## FAQ
 - What about Model-View-Intent? Does the Coordinator pattern work well with MVI too? Sure, take a look [here](http://hannesdorfmann.com/android/mosby3-mvi-8)
 - What if I don't want to use Fragments at all? How hard is it to write my own back stack that plays nice with the Coordinator pattern (i.e. just using custom ViewGroups)? Stay tuned, I'm working on a prove of concept and will share it in my blog. Hint: Finite state machines FTW.
 - Do I have to use a single Activity? No, use whatever you want to do. You can have multiple activities with multiple fragments, whatever works best for you. These implementation details are hidden behind the Navigator class
 - Do I have to have one giant Navigator class? Absolutely not! Create multiple Navigator classes (i.e. one for each flow) to keep them small and maintainable.