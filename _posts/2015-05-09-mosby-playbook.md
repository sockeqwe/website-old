---
layout: post
published: true
title: Stinson's Playbook for Mosby
mathjax: false
featured: false
comments: true
headline: Stinson's Playbook for Mosby
categories:
  - Android
tags: [android, software-architecture, design-patterns]
imagefeature: mosby/playbook.png
---

In my [previous](hannesdorfmann.com/android/mosby/) blog post I introduced Mosby, a **M**odel-**V**iew-**P**resenter library for android. This time I'm going to discuss some more details about MVP in general and how to use Mosby. I have implemented a mail client sample which can be found on [Github](https://github.com/sockeqwe/mosby/tree/master/sample-mail) which is used in this blog entry to describe how to use Mosby correctly and to answer some of the common questions I have been asked after having released Mosby.

While writing this sample app I have improved Mosby as well. I'm very happy to announce that I have released [Mosby 1.1.0](https://github.com/sockeqwe/mosby/releases/tag/1.1.0) together with this blog post. Like you already know (or have suggested) the name Mosby comes from one of my favorite tv shows: _How I met your Mother_. In this blog post I will try to frame some rules about MVP and Mosby, aligned with Barney Stinson's legen ... wait for it ... dary Playbook.

As already mentioned I have written a sample app which mimics a mail client. It's not a real mail client, there is no POP3 or IMAP Server behind it. All the data is randomly generated on app start. There is no persistent layer like local database on the device. The `APK` file can be downloaded from [here](https://github.com/sockeqwe/mosby/releases/download/1.1.0/sample-mail.apk). If you like me are to lazy to install the APK file on your phone then this video is for you:

<p>
<iframe width="640" height="480" src="https://www.youtube.com/embed/_dEYtXgoyBM?rel=0" frameborder="0" allowfullscreen></iframe>
</p>

Of course the whole app is based on Mosby. Regarding the data structure: A `Mail` is linked to a `Person` as sender and another Person as receiver. Every mail is associated to exactly one `Label`. A Label is just something like a "folder". A mail can be assigned to one Label. One Label has arbitrary many mails assigned. A Label is basically just a String. In this sample app there are four labels: "Inbox", "Sent", "Spam" and "Trash". So deleting a mail form inbox is just reassigning the label of the mail from "Inbox" to "Trash".
`MailProvider` is the central business logic element where we will query for a list of mails and so on. Furthermore, an `AccountManager` exists who is responsible to authenticate a user. As you see in the video on first app start you have to sign in to see the mails. Internally, for the business logic (`MailProvider` and `AccountManager`) I use RxJava. If you are not familiar with RxJava, don't worry. All you have to know is, that RxJava offers an `Observable` to do some kind of query and as callback the methods `onData()`, `onCompleted()` or `onError()` gets called. If you are familiar with RxJava, please DON'T look at the code! It's pretty ugly as I changed mind several times during development the model layer and didn't refactored everything as I would do for a real app. Please note that this sample app is about Mosby and not how to write a clean business logic with RxJava. Note also that even if I have used Dagger2 for dependency injection in this app is not a reference app for Dagger2 nor for material design. For sure there is room for improvements and I would gladly accept pull requests for this sample app.

As you seen in the video I'm simulating network traffic by adding a delay of two seconds to every request like loading a list of mails. I also simulate network problems: Every fifth request will fail (that's why you see the error view quite often in the video). Furthermore, I simulate authentication problems. So after 15 request the user has to sign in again (a login button will be dipslayed).

As stated in the beginning, the first Rules will be related on software architecture and MVP in general while the last ones are about Mosby.

# Rule 1: Don't "over-architect"
Take your time to make the right decision. A clean software architecture is necessary. However, at some point you have to start coding. Planning is important but in a real world project you always have to refactor your code, even after many hours of analyzing every aspect of your app. In fact refactoring is a vital part of development, you simply can't avoid it (but you can try to minimize the number of refactoring). **Don't fall into the [Analysis Paralysis](http://en.wikipedia.org/wiki/Analysis_paralysis) anti pattern.** We java developer also tend to run into a similar "problem": **Don't over-generalize everything**. From my personal experience I think that comparing with iOS Developer we android developer tend to write more clean but also over generalized code and that's one reason why developing an android app usually takes longer then developing the same app for iOS. Don't get me wrong I really appreciate clean code and design patterns. I can give you a concrete example from my experience: In one of our apps we use a kind of ImageProxy. The idea is simple: Given an image url we know that our server provides different versions of this image. Depending on the device, ImageView size, internet connection  and speed the correct version for this image should be loaded. So basically we just "rewrite" the url from _http://www.example.com/foo.jpg_ to _http://www.example.com/foo.jpg?resolution=high&width=400_. Sounds simple, doesn't it? So I have started to implement this ImageProxy library. Internally I have used Picasso for loading and caching images. However, I decided to keep it very abstract and to make components of the library modular and changeable, i.e you could replace Picasso with any other http image loading library like Glide. I also implemented the [Strategy Pattern](http://en.wikipedia.org/wiki/Strategy_pattern) for dynamically picking the right version of the image to load and to make it more extendible. The idea was that you can add more strategy implementations preconfigured and dynamically at runtime. Guess what, we have never changed the internal core for image loading (Picasso) nor have we added other strategies since we have added the ImageProxy to our app about 3 years ago (app still gets regularly major updates every month). It took me 2 days to implement this ImageProxy library, I have written 10 classes/interfaces. My coworker implementing the same thing on iOS spend about 6 hours and has written 3 classes which mainly contains some `if-else` statements. You may ask yourself how this rule is related to Mosby. Take it as a general advice. Start with doing the simple thing and afterwards refactor and make it more extensible as app and requirements grow. The same is valid while using Mosby. Try to keep your Presenters simple. Don't think about too long "how could the presenter be extended if we add feature XY in one year?".

# Rule 2: Use inheritance wisely
To say it from the very beginning: I'm not the biggest fan of inheritance. Why? Inheritance is good to add properties and functionality. Unfortunately I see many developers using it wrong. For instance you are working in a team on a certain app. You have written a MVP View that implements pull-to-refresh on a ListView. This is the base MVP View you use or extend from in almost every screen in your app. It's super handy, all you have to do to implement a new screen is to subclass from that base class and to kick in a custom adapter and you are ready to go. One day one of your teammates has to implement a ListView without pull-to-refresh. So instead of writing a completely new class many developers make the wrong decision and extend from your pull-to-refresh ListView and try to find a hacky workaround to "remove" the pull-to-refresh feature. This may lead to unmaintainable code, you may see methods overriden with empty implementation and last but not least you introduce some kind of tight coupling from pull-to-refresh ListView to the new class since you can't change the base pull-to-refresh ListView implementation without having to fear that you break the non pull-to-refresh ListView class. One possible solution your teammate should have done is to refactor the original pull-to-refresh ListView class and pull out a ListView class without pull-to-refresh and make the pull-to-refresh ListView class extend from it. However, if you decide to going this way you may end up having a large inheritance hierarchy, because every subclass adds a little feature like `B extends A` (adds feature X), `C extends B` (adds feature Y), `D extends C` (adds feature Z) and so on. You see pretty the same in the sample mail app:

  - `AuthFragment`: an LCE (Loading-Content-Error) Fragment with an additional state to display a "not authenticated - sign in" button.

  - `AuthRefreshFragment`: An extension of `AuthFragment` that uses `SwipeRefreshLayout` as main content view. So one could say: "It's a AuthFragment with pull-to-refresh support".

  - `AuthRefreshRecyclerFragment`: Puts a `RecyclerView` into the `SwipeRefreshLayout`.

  - `BaseMailsFragment`: An abstract class to displays a list of mails (`List<Mail>`) by using the `RecyclerView`

  - There are serval concrete classes that extend from `BaseMailsFragment` like `MailsFragment`, `SearchFragment` and `ProfileMailsFragment`

 The inheritance hierarchy might be correct, but do you really think that someone who recently joined the team who is developing the Mosby mail app would pick the right base class to implement a new feature XY? I doubt that I would pick the right class (and I am the author of this inheritance hierarchy) for the new feature XY. In java it's all about interfaces and so it should be on android. I, personally, prefer interfaces and delegation over inheritance.

 # Rule 3: Don't see MVP as an MVC variant
 Some people find it hard to understand what the Presenter exactly is, if you say to try to explain that MVP is a variant of MVC (Model-View-Controller). Especially iOS developer having a hard to understand the difference of Controller and Presenter because the "grew up" with the fixed idea and definition of an iOS controller. From my point of view MVP is not a variant of MVC but it wraps around MVP. Take a look at your MVC powered app. Typically you have your View and a Controller (i.e. a `Fragment` on Android  or `UIViewController` on iOS) which handles click events, binds data and observers ListView (or implements a `UITableViewDelegate` for `UITableView` on iOS`) and so on. If you have this picture in mind now take a step back and try to imagine that the controller is part of the view and not connected directly to your model (business logic). The presenter sits in the middle between controller and model like this:

 ![MVP with Controller](/images/mosby/mvp-controller.png)

 I want to explain that with an example I already used in my previous blog post. In this example you you want to display a list of users queried from a database. The action starts when the user clicks on the load button, displays a ProgressBar while querying the database (async) and displays the ListView with the items afterwards. The workflow looks like this:

 ![MVP with Controller](/images/mosby/mvp-workflow.png)

In my opinion the **Presenter does not replace the controller** rather the presenter "controlls" the View which the Controller is part of. The controller is the component that handles the click event and forwards it to the presenter. The controller is the responsible component to coordinate animations like hiding ProgressBar and displaying ListView instead. The controller is listening for scroll events on the ListView to do some parallax item animations or scroll the toolbar in and out while scrolling the ListView. So all that UI related stuff still gets controlled by the controller and **not by the presenter**. The presenter is responsible to coordinate the overall state of the view layer (composed of UI components and controller). So it's the job of the presenter to tell the view layer that the loading animation should be displayed now or that the ListView should be displayed now because the data is ready to be displayed. A good example can be seen in the workflow image above: After the user has clicked on the “load user button” (step 1) the view doesn’t show the loading animation (ProgressBar) directly. It’s the presenter (step 2) who explicitly tells the view to show the loading animation.

By the way: Yes, I think that MVP suits quite well on iOS as well!

# Rule 4: Take separation of Model, View and Presenter serious
 That might be obvious but to achieve a clean, modular, reusable and maintainable codebase you should really look over your code and ask yourself what that line of code is doing. Is that line of code related to View (UI component or UI controller like OnClickListener) or doing presenters job (coordinating the views state) or business logic (i.e. loading data). The following (pseudeo) code snipped shows a [BLOB](http://www.antipatterns.com/briefing/sld024.htm) Fragment (all things in one huge class) that displays a form to do a login (similar to the mail sample login):

 {% highlight java %}
 public class LoginFragment extends Fragment {

   @InjectView(R.id.username) EditText username;
   @InjectView(R.id.password) EditText password;
   @InjectView(R.id.loginButton) ActionProcessButton loginButton;
   @InjectView(R.id.errorView) TextView errorView;
   @InjectView(R.id.loginForm) ViewGroup loginForm;

   AsyncTask loginTask;
   AccountManager accountManager;

   public void onCreateView(LayoutInflater inflater, ViewGroup root, Bundle b){
     return inflater.inflate(R.layout.fragment_login, root, false);
   }

   @OnClick(R.id.loginButton) public void onLoginClicked() {


     // Controll UI components --> Controllers job
     errorView.setVisibility(View.GONE);
     loginForm.clearAnimation();

     doLogin();
   }

   private void doLogin(){

     // Coordinates the state of the view --> Presenters job
     showLoading();


     loginTask = new AsyncTask(){

       // returns true if login successful, otherwise false
       protected boolean doInBackground(){

         // Interacts with Business logic --> Presenters job
         User user = accountManager.login(username, password);
         return user != null;
       }

       protected void onPostExecute(Boolean successful) {

         // Coordinates the state of the view --> Presenters job
          if (successful){
            loginSuccessful();
          } else {
            showError();
          }
        }
      }.execute();

    }

    private void showError(){

     // Manipulate the UI --> Controllers job
     loginForm.clearAnimation();
     Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
     loginForm.startAnimation(shake);
     loginButton.showLoading(false);

     errorView.setVisibility(View.VISIBLE);
    }

    @Override public void showLoading() {

    errorView.setVisibility(View.GONE);
    loginButton.showLoading(true);
  }


  @Override public void loginSuccessful() {
    getActivity().finish();
  }

 }
 {% endhighlight %}

 I already have added comments with pointing out responsibilities in MVP. Lets refactor that single huge class into Model, View, Presenter. The Model is the `AccountManager`. The view should have no knowledge of the model, so pull that out. Everything that coordinates the views state should be moved into the presenter. So AsyncTask method must to be moved to the Presenter. The presenter must be able to coordinate the view. Therefore we introduce a interface for the view `LoginView` where we define the methods `showError()` `loginSuccessful()` and `showLoading()`. We do that with an interface because we want to keep the Presenter as platform independent as possible. Furthermore it makes writing unit test more easy (read more about that in my previous blog post). As we have discussed in the previous rule the Fragment is a Controller and therefore part of the View. So the fragment at the end should only contain code controlling UI components. The refactored code (with Mosby View State support) looks like this:

 {% highlight java %}
 public class LoginFragment extends MvpViewStateFragment<LoginView, LoginPresenter>
    implements LoginView {

  @InjectView(R.id.username) EditText username;
  @InjectView(R.id.password) EditText password;
  @InjectView(R.id.loginButton) ActionProcessButton loginButton;
  @InjectView(R.id.errorView) TextView errorView;
  @InjectView(R.id.loginForm) ViewGroup loginForm;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setRetainInstance(true);
  }

  @Override protected int getLayoutRes() {
    return R.layout.fragment_login;
  }

  @Override public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    super.onViewCreated(view, savedInstanceState);

    int startDelay = getResources().getInteger(android.R.integer.config_mediumAnimTime) + 100;
    LayoutTransition transition = new LayoutTransition();
    transition.enableTransitionType(LayoutTransition.CHANGING);
    transition.setStartDelay(LayoutTransition.APPEARING, startDelay);
    transition.setStartDelay(LayoutTransition.CHANGE_APPEARING, startDelay);
    loginForm.setLayoutTransition(transition);
  }

  @Override public ViewState createViewState() {
    return new LoginViewState();
  }

  @Override public LoginPresenter createPresenter() {
    return new LoginPresenter();
  }

  @OnClick(R.id.loginButton) public void onLoginClicked() {

    // Check for empty fields
    String uname = username.getText().toString();
    String pass = password.getText().toString();

    loginForm.clearAnimation();

    // Start login
    presenter.doLogin(new AuthCredentials(uname, pass));
  }

  @Override public void onNewViewStateInstance() {
    showLoginForm();
  }

  @Override public void showLoginForm() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoginForm();

    errorView.setVisibility(View.GONE);
    setFormEnabled(true);
    loginButton.setLoading(false);
  }

  @Override public void showError() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowError();

    loginButton.setLoading(false);

    if (!isRestoringViewState()) {
      // Enable animations only if not restoring view state
      loginForm.clearAnimation();
      Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
      loginForm.startAnimation(shake);
    }

    errorView.setVisibility(View.VISIBLE);
  }

  @Override public void showLoading() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoading();
    errorView.setVisibility(View.GONE);
    setFormEnabled(false);

    loginButton.setLoading(true);
  }

  private void setFormEnabled(boolean enabled) {
    username.setEnabled(enabled);
    password.setEnabled(enabled);
    loginButton.setEnabled(enabled);
  }

  @Override public void loginSuccessful() {
    getActivity().finish();
  }

}
{% endhighlight %}

As you can see `LoginFragment` now contains only code related to UI components.

The presenter looks like this (using RxJava instead of AsyncTask):

{% highlight java %}
public class LoginPresenter extends MvpBasePresenter<LoginView> {

  private AccountManager accountManager;
  private Subscriber<Account> subscriber;

  @Inject public LoginPresenter(AccountManager accountManager) {
    this.accountManager = accountManager;
  }

  public void doLogin(AuthCredentials credentials) {

    if (isViewAttached()) {
      getView().showLoading();
    }

    // Kind of "callback"
    subscriber = new Subscriber<Account>() {
      @Override public void onCompleted() {
        if(isViewAttached()){
          getView().loginSuccessful();
        }
      }

      @Override public void onError(Throwable e) {
        if (isViewAttached()){
          getView().showError();
        }
      }

      @Override public void onNext(Account account) {
      }
    };

    // do the login
    accountManager.doLogin(credentials).subscribe(subscriber);
  }

}
{% endhighlight %}

So instead of having one huge Fragment with over 300 lines of code now we have two classes with about 100 lines of code each. Moreover, now we have reusable, maintainable code and a clear separation of responsibilities.

# Rule 5: Navigation is a UI thing
 In some other MVP libraries like single page html sides (javascript) it's the presenter who gets instantiated as fist compoenent and the presenter creates the view. This is not the case on Android (or at least not if you use Mosby) since the Android Framework defines Activity as entrypoint and Actiyity is seen as part of the View layer. So it doesn't make sense to let the Presenter start the Intent for launching a new Activity.

# Rule 6: What about onActivityResult()
 First try to avoid `onActivityResult()` for inner app communication. If you just want to propagate a result from one activity to antoher then an `EventBus` might by a better choice. However, there a  situation where you have to use `onActivityResult()` for instance if you want to communicate with other apps like the cammera app. Should you forward the result to the presenter? It depends: If you simply want to display the taken foto as it is in an `ImageView` then there is no need to forward that to the Presenter. Remember, the presenter is responsible to coordinate the state. So displaying just that Image in a ImageView most of the time doesn't change the state and can be done directly.

 {% highlight java %}
 @Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        imageView.setImageBitmap(imageBitmap);
    }
}
{% endhighlight %}

If you want do some additional image processing then you would do that async. In that case you should use the presenter and let the presenter controll the view (like showing a loading view while processing image) like this:

{% highlight java %}
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
   if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
       Bundle extras = data.getExtras();
       Bitmap imageBitmap = (Bitmap) extras.get("data");
       presenter.makeThumbnail(imageBitmap);
   }
}
{% endhighlight %}


# Rule 5: MVP and EventBus: A match made in heaven
 An `EventBus` allows you to inter-communicate between decoupled components by posting and receiving  events. I assume that you already are familar with the concept of an `EventBus`. EventBus is not MVP related. I first came in touch with MVP and EventBus with GWT in late 2009 and it was just mind blowing. From now on I never used MVP without EventBus anymore. Of course I still use EventBus quite a lot in Android. With RxJava some things previously done by an EventBus has been replaced with RxJava's observer "push updates mechanism" (have a look at [SqlBrite](https://github.com/square/sqlbrite) for a concrete use case). The difference between RxJava's push mechanism  and EventBus is that in RxJava you explicitly have to subscribe on an observable and you need to have a reference to that observable wheras an EventBus is a little bit more loosely coupled since you just have to know the event you are listening for. So when I'm talking here about an EventBus you can think of every other push based update mechanism as well.

 I see two use cases for Eventbus:

   1. **EventBus for business logic events:** The Events related to the business logic are posted to the EventBus. As we have discussed previously the View shouldn't have any knowledge of the business logic. Therefore, the presenter is the component that listens for business logic events through the Eventbus. In Mosby you should regitster and unregister the presenter from EventBus in `attachView()` and `detachView()` respectively. Why not registering in the constructor of the Presenter? Because if you have retaining Views (like retaining Fragment) the Presenter gets created only once, but the view still gets attached and reattached to the presenter during orientation changes.

  In the mail sample I use EventBus quite a lot. The first thing you may have noticed in the video is that after the login was successful all views automatically beginn to load data. You can see this even better if you start the mail app on a tablet.

  <p>
  <iframe width="640" height="480" src="https://www.youtube.com/embed/d898QONavBo?rel=0" frameborder="0" allowfullscreen></iframe>
  </p>

 The `LoginPresenter` fires a `LoginSuccessfulEvent` to inform all other presenters that the user has been logged in successfully. The presenters will then trigger a reload of the view.


![Inbox-States](/images/mosby/login-eventbus.jpg)


  2. **EventBus for UI Events:** Another usecase is to use the EventBus in your UI or to communicate between Fragments and Activities. The official [docs recommandation](http://developer.android.com/training/basics/fragments/communicating.html) to do a communicate between fragments and the host activity is to define a listener interface and cast the Activity in fragments `onAttach()` method to the listener:

 {% highlight java %}
  @Override
  public void onAttach(Activity activity) {
      super.onAttach(activity);

      // This makes sure that the container activity has implemented
      // the callback interface. If not, it throws an exception
      try {
          mCallback = (OnHeadlineSelectedListener) activity;
      } catch (ClassCastException e) {
          throw new ClassCastException(activity.toString()
                  + " must implement OnHeadlineSelectedListener");
      }
  }
{% endhighlight %}

 I don't like that (casting an object to an interface). Every other java developer, i.e. in the J2EE world, would throw his hands up in horror if he sees something like this in his code. So why should it be the recommended way (btw. on iOS you are doing pretty the same)? I usually use an EventBus for doing that. The fragment fires an Event on the EventBus and the parent Activity is subscribed for that Event on the EventBus. The advantage of an EventBus is even bigger, if you have a splitpane layout displaying Fragment A on the left and Fragment B on the right and Fragment A wants to communicate with Fragment B. The [docs](http://developer.android.com/training/basics/fragments/communicating.html) says that in that case Fragment A should talk to the activity and the activity forwards the information to Fragment B. With an EventBus Fragment A can fire an Event and Fragment B, who is regitstered for that Event on the EventBus, gets notified immediately without having the Acticity as middle man. In that way all 3 components (Acticity, Fragment A  and Fragment B) are decoupled (no one has a reference to each other).

 A common use case where EventBus is useful is when implementig navigation drawer. If you click on a menu item the Activity should replace the current main fragment with the other fragment associated with the clicked menu item. It turns out that Android itself already offers an EventBus you already use: `Intents`. Instead of using an external EventBus you could use Intent like I did in the mail sample. I set `MainActivity` to `android:launchMode="singleTop"` and check in `onNewIntent()` for the navigation events. The advantage with Intents is that this kind of Navigation can be used system wide (and not app wide like EventBus). For instance when a new Mail is received a Notification will be shown in the StatusBar. Clicking on the Notification would open the Mail to read it. There could be a Button in Notification that opens the Inbox by clicking on it. If you use a EventBus for navigation in your app, then you might get in the problem that you have to deal with intent and with Event fired through EventBus. You should think about that scenario. If your app can not run into this scenario then using an EventBus for navigation is fine as well.

 # Rule 6: Optimistic propagation
 Let's have a look at the mail app. When you star or unstar a Mail the star icon changes immediately. However, saving that a mail has changed takes two seconds and can fail. For a better user experience we say we are optimistic and assume that changing the stared mail will be successful. It gives the user of your app an immediately feedback and the feeling that your app works lightning fast.

 <p>
 <iframe width="640" height="480" src="https://www.youtube.com/embed/7HPtTg35ZS0?rel=0" frameborder="0" allowfullscreen></iframe>
 </p>

 At first glance you may think that this is too complex to implement for your simple app you are currently working on. It seems that you have to respect so many things for implementig that. Don't worry, with a clean architecture like MVP and an EventBus it's quite easy:

 {% highlight java %}
 public class BaseRxMailPresenter<V extends BaseMailView<M>, M>
     extends BaseRxAuthPresenter<V, M> {

   @Inject public BaseRxMailPresenter(MailProvider mailProvider, EventBus eventBus) {
     super(mailProvider, eventBus);
   }

   public void starMail(final Mail mail, final boolean star) {

     // optimistic propagation
     if (star) {
       eventBus.post(new MailStaredEvent(mail.getId()));
     } else {
       eventBus.post(new MailUnstaredEvent(mail.getId()));
     }

     mailProvider.starMail(mail.getId(), star)
         .subscribe(new Subscriber<Mail>() {
           @Override public void onCompleted() {
           }

           @Override public void onError(Throwable e) {
             // Oops, something went wrong, "undo"
             if (star) {
               eventBus.post(new MailUnstaredEvent(mail.getId()));
             } else {
               eventBus.post(new MailStaredEvent(mail.getId()));
             }

             if (isViewAttached()) {
               if (star) {
                 getView().showStaringFailed(mail);
               } else {
                 getView().showUnstaringFailed(mail);
               }
             }
           }

           @Override public void onNext(Mail mail) {
           }
         });

     // Note: that we don't cancel this operation in detachView().
     // We want to ensure that this operation finishes
   }

  ...

 }
{% endhighlight %}

 If we star a Mail we fire `MailStaredEvent()` and if we unstar a mail we fire a `MailUnstaredEvent`. As you seen in the video by staring a mail the left pane displaying the list of mails gets updated as well since the presenter is registered on those events. The trick is that we fire the corresponding event even before we make the corresponding business logic call and we use the counterpart event to "undo" it in case that that the business logic call fails. Managing that logic in the Presenter is quite easy as you have seen in the code snipped above. But we also have to update the view. In MVP we have this clear separation between View and business logic, therefore the presenter is responsible to update the view by calling the corresponding method defined in the view's interface. As you see with a clean MVP architecture and EventBus doing such "complex" things is extremly easy. Presenter is doing the business logic staff and then the code in your View implemetation is just a one-liner. However, if you would do that all in your Fragment (without MVP) then yes, it might be a little bit more complicated to maintain the state.

 Please note that we are not canceling this business logic call in `presenter.detachView()` because we want to ensure that this call finishes. You may ask yourself now if that doesn't result in memory leaks during orientation changes. The answer is "it depends on the definition of memory leak". View get's detached from presenter during screen orientation changes. So the huge memory consuming view objects like Activity or fragment don't leak memory. However, the Presenter creates a new anonymous Subscriber object. Every anonymous object has a hard reference to the surrounding object. So yes, the presenter can't be garbage collected as long as Subscriber gets unsubsribed (business logic call has finished). But I wouldn't call this a memory leak. It's kind of a temporarly and wanted memory leak and from my point of view completely fine since presenter is a lightweight object and you know that the presenter can be garbage collected after the business call has finished.

 Another sample where optimic propagation improves the user experience of your app is when moving a mail from "Inbox" to "Spam":

 <p>
 <iframe width="640" height="480" src="https://www.youtube.com/embed/rA22tsGdeY4?rel=0" frameborder="0" allowfullscreen></iframe>
 </p>


# Rule 7: MVP scales
 Most of the time you have have one View (and Presenter) filling the whole screen. But that doesn't has to be the case. Typically Fragments are a good candidate for splitting your screen into modular decoupled Views with his own Presenter. The sample mail client implents this as well when you open a profile by clicking on the senders image (avatar):

<p>
<iframe width="640" height="480" src="https://www.youtube.com/embed/4Mce1ljHJz0?rel=0" frameborder="0" allowfullscreen></iframe>
 <p>

 The activity itself is a View having a ViewPager as content view. The Presenter loads a List of `ProfileScreen`. A ProfileScreen is just a POJO.

 {% highlight java %}
 public class ProfileScreen implements Parcelable {

  public static final int TYPE_MAILS = 0;
  public static final int TYPE_ABOUT = 1;

  int type;
  String name;

  private ProfileScreen() {
  }

  public ProfileScreen(int type, String name) {
    this.type = type;
    this.name = name;
  }

  public int getType() {
    return type;
  }

  public String getName() {
    return name;
  }
}
{% endhighlight %}

 The ViewPager Adapter takes the `List<ProfileScreen>` and instantiates concrete fragments:

 {% highlight java %}
 public class ProfileScreensAdapter extends FragmentPagerAdapter {


  @Override public Fragment getItem(int position) {

    ProfileScreen screen = screens.get(position);
    if (screen.getType() == ProfileScreen.TYPE_MAILS) {
      return new ProfileMailsFragmentBuilder(person).build();
    }
    if (screen.getType() == ProfileScreen.TYPE_ABOUT) {
      return new AboutFragmentBuilder(person).build();
    }

    throw new RuntimeException("Unknown Profile Screen (no fragment associated with this type");
  }

  @Override public CharSequence getPageTitle(int position) {
    return screens.get(position).getName();
  }
}
{% endhighlight %}

Loading `List<ProfileScreen>` takes 2 seconds (simulates loading the screens for the viewpager dynamically from a backend). So basically we have a LCE (Loading-Content-Error) and therfore we can use Mosby's `MvpLceViewStateActivity`. Next every Fragment in the ViewPager is a MVP View and hast it's own presenter. I guess you get the overall picture: A MVP View can contain indipendent MVP Views.

# Rule 8: Not every screen needs MVP
 This might be obvious but once you are in the "I'm a super software architect" mode you may forget about that there are screens and informations that just display static content. Does content does not need MVP.

# Rule 9: Only display one Model per MVP view
 Mosby assumes that every MVP view displays exactly one model (Note that displaying a list of items is still displaying one model, the list). Why? Because dealing with differnt models in the same view increases complexity dramatically and we clearly want to avoid spaghetti code. In the sample app I faced that problem in the menu. I haven't refactored that yet to give you the possiblity to understand with a concrete code sample. Have a look at the screenshot of the menu:

 ![Menu](/images/mosby/mail-menu.png)

 The header displays the current authenticated user, represented as `Account`, while the clickable menu items is a `List<Label>` (just ignore statistics). So we have two Models displayed in the same MVP View namely `MenuFragment`. The problem is that it makes it your code more complex and harder to maintain when you work with two Models in the same view. Things get even more complex if you decide to use Mosby's ViewState. Because, now you not only have to store the state showing Labels, loading labels and error while loading labels, no you also have to store that you the user is authenticate or not.

 The solution to this problem is to split the one big MVP View into two views with its own Presenter and own ViewState:

 ![Menu](/images/mosby/menu-refactored.png)


# Rule 10: Use LCE only if you have LCE Views
LCE stands for Loading-Content-Error. Mosby ships with `MvpLceViewStateFragment implements MvpLceView` which displays either the content (like a list of mails) or loading (like a progressbar) or an error view (like a TextView displaying an error message). This might be handy, because you don't have to implement the switching of displaying content, displaying error, displaying loading by your own, but you shouldn't use it as base class for everything. Only use it if your view can reach all three view states (showing loading, content and error). Have a look at the sign in view:

{% highlight java %}
public interface LoginView extends MvpView {

  // Shows the login form
  public void showLoginForm();

  // Called if username / password is incorrect
  public void showError();

  // Shows a loading animation while checking auth credentials
  public void showLoading();

  // Called if sign in was successful. Finishes the activity. User is authenticated afterwards.
  public void loginSuccessful();
}
{% endhighlight %}

At first glance you might assume that `LoginView` is a `MvpLceView` with just having `showContent()` renamed to `showLoginForm()`. Have a closer look at `MvpLceView` definition:

{% highlight java %}
public interface MvpLceView<M> extends MvpView {

  public void showLoading(boolean pullToRefresh);

  public void showContent();

  public void showError(Throwable e, boolean pullToRefresh);

  public void setData(M data);

  public void loadData(boolean pullToRefresh);
}
{% endhighlight %}

Methods like `loadData()` and `setData()` are not needed in `LoginView` nor make it sense to have pull-to-refresh support. You could simply implement that methods with an empty implementation. Defining an interface is like making a contract with other software components. Your interface promises that the method exists, but doing nothing on invoking this method is a violation of the contract. Also your code gets harder to understand and maintain if you have plenty of methods that get called from some other components but are doing nothing. Furthermore, if you use `MvpLceView` you tend to use a already existing `MvpLcePresenter` implementation as base class for `LoginPresenter`. The problem is that `MvpLcePresenter` is "optimized" for `MvpLceView`. So you may have to implement some workarounds in your presenter to achive what you want to do. Simply avoid that by not using LCE related classes if your view doesn't have full LCE support.

# Rule 11: Writing custom ViewStates
Mosby has a mechanism to save and restore the views state which is useful to handle orientation changes. Some ViewState implementations for `MvpLceView` are already provided by Mosby. However `LoginView` is not an `MvpLceView`. Writing a custom view state is easy:

{% highlight java %}
public class LoginViewState implements ViewState<LoginView> {

  final int STATE_SHOW_LOGIN_FORM = 0;
  final int STATE_SHOW_LOADING = 1;
  final int STATE_SHOW_ERROR = 2;

  int state = STATE_SHOW_LOGIN_FORM;

  public void setShowLoginForm() {
    state = STATE_SHOW_LOGIN_FORM;
  }

  public void setShowError() {
    state = STATE_SHOW_ERROR;
  }

  public void setShowLoading() {
    state = STATE_SHOW_LOADING;
  }

  /**
   * Is called from Mosby to apply the view state to the view.
   * We do that by calling the methods from the View interface (like the presenter does)
   */
  @Override public void apply(LoginView view, boolean retained) {

    switch (state) {
      case STATE_SHOW_LOADING:
        view.showLoading();
        break;

      case STATE_SHOW_ERROR:
        view.showError();
        break;

      case STATE_SHOW_LOGIN_FORM:
        view.showLoginForm();
        break;
    }
  }

}
{% endhighlight %}

We simply store the current view state as an integer in `state` (you could also use enum). The method `apply()` get's called from Mosby and this is the point where we restore the view state to the associated view.
You may wonder how the ViewState is connected to your Fragment or Activity. `MvpViewStateFragment` has a method `createViewState()` called by Mosby internally, which you have to implement. You just have to return a `LoginViewState` instance. However, you have to set the LoginViewState internal state by hand. Typically you do that in the methods defined by `LoginView` interface as shown below:
{% highlight java %}

public class LoginFragment extends MvpViewStateFragment<LoginView, LoginPresenter>
    implements LoginView {

  @Override public void showLoginForm() {

    // Set View state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoginForm();

    errorView.setVisibility(View.GONE);

    ...
  }

  @Override public void showError() {

    // Set the view state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowError();

    if (!isRestoringViewState()) {
      // Enable animations only if not restoring view state
      loginForm.clearAnimation();
      Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
      loginForm.startAnimation(shake);
    }

    errorView.setVisibility(View.VISIBLE);

    ...
  }

  @Override public void showLoading() {

    // Set the view state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoading();

    errorView.setVisibility(View.GONE);

    ...
  }
}
{% endhighlight %}

Sometimes you have to know if the method get's called from presenter or because of the restoring the view state, typically when working with view animations. You can check that with `isRestoringViewState()` like `showError()` does (see above).

#Rule 12: Testing custom ViewState
Since `LoginView` and `LoginViewState` are plain old java classes with no dependencies to the android framework, you can simply write plain old java unit test for `LoginViewState`:

{% highlight java %}
@Test
public void testShowLoginForm(){

  final AtomicBoolean loginCalled = new AtomicBoolean(false);
  LoginView view = new LoginView() {

    @Override public void showLoginForm() {
      loginCalled.set(true);
    }

    @Override public void showError() {
      Assert.fail("showError() instead of showLoginForm()");
    }

    @Override public void showLoading() {
      Assert.fail("showLoading() instead of showLoginForm()");
    }

    @Override public void loginSuccessful() {
      Assert.fail("loginSuccessful() instead of showLoginForm()");
    }
  };

  LoginViewState viewState = new LoginViewState();
  viewState.setShowLoginForm();
  viewState.apply(view, false);
  Assert.assertTrue(loginCalled.get());

}
{% endhighlight %}

The idea is simple: We create a `LoginViewState` and set the internal state `viewState.setShowLoginForm()`. Next we create a mock view and call the `apply()` method on the view state. Then we check if the expected method has been called. You could also mock the LoginView with mocking libraries like Mockito, but I wanted to keep the sample simple and independent from other third party libraries.

To test if `LoginFragment implements LoginView` restores it's state correctly during screen orientation changes it's enough to test the ViewState because you can assume that Mosby's internal are working correct.

So far we only have tested if restoring works correctly. We also have to test if we are setting the view state correctly. However, that is done in `LoginFragment`, so we have to test the Fragment rather than the `LoginViewState`. Thanks to Robolectric this is also straightforward:
{% highlight java %}
@Test
public void testSettingState(){
  LoginFragment fragment = new LoginFragment();
  FragmentTestUtil.startFragment(fragment);
  fragment.showLoginForm();

  LoginViewState vs = (LoginViewState) fragment.getViewState();

  Assert.assertEquals(vs.state, vs.STATE_SHOW_LOGIN_FORM);
}
{% endhighlight %}

We simply call `showLoginForm()` and we check if the ViewState is `STATE_SHOW_LOGIN_FORM` after this call.

You could also test if `LoginFragment` handles screen orientation changes correctly by writing instrumentation test (i.e. with espresso), but the point is that in Mosby you have this clear separation between view and view state and therefore you can test the view state independently.


# Rule 13: ViewStates and ViewState variants
You may get confused what exactly a `ViewState` is. A `ViewState` is a state that the view can reach. For instance showing a loading view (ProgressBar) instead of the content view (ListView displaying items). That are clearly two different view states: the view should display either loading view or content view. But what if we add pull-to-refresh support to the content view (SwipeRefreshLayout around ListView). If the user triggers a pull-to-refresh action we show the `ListView` (content view) and a `ProgressBar` at the same time. The question is: Which state is that? The view always is in exactly one state (that's the definition of view state). So during pull-to-refresh the view is not in showing content and showing loading at the same time. Internally the view is still in exactly one state. There are two possibilities:
 1. Introduce a new state for pull-to-refresh (displaying ListView and ProgressBar at the same time).
 2. Use "show loading view state": Additionally we store the information that a pull-to-refresh was triggered (i.e. a boolean flag) and that the ListView should be displayed as well. Using an exisiting ViewState and alter that one with additional info (i.e. a boolean flag for pull-to-refresh) is called a **Variant of ViewState**, because we are altering the view state by adding the boolean pull-to-refresh flag to have two variants of the "show loading view state". The first variant is where pull-to-refresh flag is true (view should display ListView and pull-to-refresh indicator) and the second variant is where pull-to-refresh flag is false (view should display only a ProgressBar).

 It doesn't make a difference which one of this two options you use, it's just an internal implementation detail. The point with ViewState is, that there is exactly one view state the view is in. However, one view state can have few variants. Having variants may require multiple information to be stored and multiple view methods to be invoked to restore this view state variant. Mosby's default MvpLceViewState implementation uses appraoch number 2 which looks like this:
 {% highlight java %}

 private boolean pullToRefresh;

 @Override
 public void apply(V view, boolean retained) {
   if (currentViewState == STATE_SHOW_LOADING) {

    if (pullToRefresh) {
      view.setData(loadedData);
      view.showContent();
    }

    // view displays pull-to-refresh indicator if pullToRefresh == true,
    // otherwise hides content view or error view and displays big ProgressBar instead
    view.showLoading(pullToRefresh);
  }

  ...
}
 {% endhighlight %}

 However, you shouldn't have more than two variants of a ViewState and only prefer variants over defining a new view state if a variant is simple and doesn't not require complex restoring (in `apply()`).

Alright, now you should have an idea what the difference between a view state (i.e. show loading) and a view state variant (i.e. show loading with pull-to-refresh) is.

Let's have a look at the mail sample app. Here we can find some custom view states that extend from already existing ones. `AuthParcelableDataViewState` for instance is an extension of  `LceViewState` and has four states:

![Inbox-States](/images/mosby/inbox_states.jpg)

Adding an additional state is quite easy. Internally `LceViewstate` (from which `AuthParcelableDataViewState` inherits) has a field `int state` that is used for saving the current state. We simply use that field to add a new state for the case the user is not authenticated as shown below:

{% highlight java %}
public class AuthParcelableDataViewState<D extends Parcelable, V extends AuthView<D>>
    extends ParcelableDataLceViewState<D, V> implements AuthViewState<D, V> {

  public static final int SHOWING_AUTHENTICATION_REQUIRED = 2;

  @Override public void apply(V view, boolean retained) {

    if (currentViewState == SHOWING_AUTHENTICATION_REQUIRED) {
      view.showAuthenticationRequired();
    } else {
      // call super to handle the states showing loading or error or content
      super.apply(view, retained);
    }
  }

  @Override public void setShowingAuthenticationRequired() {
    currentViewState = SHOWING_AUTHENTICATION_REQUIRED;

    // Delete any previous stored data of other view states
    loadedData = null;
    exception = null;
  }
}
{% endhighlight %}

An example of how to implement ViewState variants can be found in `SearchFragment implements SerachView` to implement pagination: Basically `SearchView` displays a list of mails, loading (with and without pull-to-refresh) and displaying an error view. Therefore we can use a LCE based view state implementation. If we want to add pagination (displaying a list of mails in chunks, i.e. display 20 mails and if the user scrolls to the end of the list then load the next 20 mails). The  first question if we want to add a ViewState Variant is which ViewState should we add the variant to the "show loading" or "show content" one? In that case it depends a little bit on your implementation. What I did to display "loading more" is I added an additional ViewType to `SearchResultAdapter` which displays the list of mails matching search criteria in the recycler view (content view). So the last item the RecyclerView will display is a row displaying a ProgressBar. Since triggering a "load more action" requires to scroll the content view (recycler view) it feels more natural to me to add the view state variant to "showing content" instead of "showing loading" (which is used for pull-to-refresh) because I have implemented the "load more indicator" as part of the RecyclerViews (content view) dataset. The `SearchViewState` with the variant simply adds a boolean flag `loadingMore`:

{% highlight java %}
// AuthCastedArrayListViewState is a LCE ViewState with an additional "not authenticated" state
public class SearchViewState extends AuthCastedArrayListViewState<List<Mail>, SearchView> {

  boolean loadingMore = false;

  public void setLoadingMore(boolean loadingMore) {
    this.loadingMore = loadingMore;
  }


  @Override public void apply(SearchView view, boolean retained) {

    super.apply(view, retained);

    if (currentViewState == STATE_SHOW_CONTENT) {
      view.showLoadMore(loadingMore);
    }

  }
}
{% endhighlight %}


If you don't feel comfortable with ViewState Variants then simply don't use Variants but create new ViewStates instead. Defining new view states is cleaner, however may requires to write a little bit more code. As I'm a lazy guy I wanted to point out that variants can be used but it's not a solution for everything and everybody.

# Rule 14: Not every UI representation is a ViewState
What if you open the inbox which is empty. Then the sample mail app desplays that:
![Inbox-States](/images/mosby/no_mails_state.png)

Is that another view state? No, it's just the view state "showing Content" with an empty List as data. The empty list will be displayed like that in the UI. The UI is allowed to represent the same view states in different ways.

# Rule 15: Not every View needs a ViewState
I'm sure that you find Mosby's concept of `ViewState` one of the most comfortable features. Handling orientation changes works like a charm out of the box. However, that comes with a price you have to pay. The ViewState gets stored into the Bundle used in `onSaveInstanceState()`. This bundle is limited in size to 1 MB. If your View displays a huge list, then you might run over this limit of 1 MB. If you use retaining Fragment you don't run into this kind of problem because the view state is stored in memory and not in the Bundle. Another thing you have to keep in mind is that  an Activity can be destoryed by the Android OS and restored if you navigate back to the Activity. When coming back the ViewState gets restored and the activity may display old and stale data, because the same data has been edited in another Activity of your app. Ususally this doesn't happen very often. However, if you display sensitive data you should keep that in mind and in that case it may be better to not use ViewState.

# Rule 15: Avoid Fragments on the backstack and child fragments
I'm sure all of you have heared about how bad fragments lifecycle are and how terrible it is to work with them. Unfortunately many arguments against Fragments are correct. However, I think I can work with fragment as long as I don't put Fragment's on backstack or use child fragments (fragments in fragments). Mosby can deal with them and makes life easier for you. But in general it's a good idea to avoid problems from the very beginning if you know that there may be trouble.


# Rule 16: Mosby supports MVP ViewGroups
If you simply want to avoid Fragments in general you can do that. Mosby offers the same MVP scaffold as for Activities and Fragments for ViewGroups as well, inclusive View State. So the API is the same for Activity, Fragment and ViewGroups. Some default implementation are already available like `MvpViewStateFrameLayout`, `MvpViewStateLinearLayout` and `MvpViewStateRelativeLayout`.
This ViewGroup implementation are also useful if you want to have subviews in fragment (since you should avoid child fragment). The mail sample uses that when you want to reassign the label of a mail:

<p>
<iframe width="640" height="480" src="https://www.youtube.com/embed/AWt-JhWi3lo?rel=0" frameborder="0" allowfullscreen></iframe>
</p>

This "Button" for assigning a Label is just a `MvpViewStateLinearLayout`. It animates the ProgressBar (loadingView) in and out while loading and uses a `ListPopupWindow` (contentView) to display the list of labels and uses a Toast to inform about errors.
