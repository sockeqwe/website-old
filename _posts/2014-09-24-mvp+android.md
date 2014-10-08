---
layout: post
published: false
title: MVP + Android
mathjax: false
featured: false
comments: true
headline: MVP + Android <br /> A match made in heaven
categories:
  - android
tags: [android, software-architecture, appkit, mvp]
---

Nowadays there are many challenges a Android developer has to face. Amongst others developers have to build modular reusable components that can be used both on mobile phones and tablets, to keep applications lifecycle in mind, think about screen orientation changes and operate and communicate with services. To build a successful app it is necessary to make the right decisions from the very beginning. Therefore, a well conceived software architecture is essential. In this blog I want to discuss the Model-View-Presenter (MVP) architecture on Android and I want to introduce **AppKit** a MVP library for android.


# Model-View-Presenter

The Model-View-Presenter design pattern is a modern pattern to seperate the view from the underlying model. MVP is a derivative of the model–view–controller (MVC) software pattern, also used mostly for building user interfaces.

![Model-View-Presenter](/images/mvp_normal.png)

* The **model** is the data (or a component to retrieve data) that will be displayed in the view (user interface).
* The **view** is an interface that displays data (the model) and routes user commands (events) to the presenter to act upon that data. The view usually  has a reference to its presenter.
* The **presenter** is the "middle-man" (played by the controller in MVC) and has references to both, model and view. Normally the presenter observes the model to receive model updates. So when an model update has happend, the presenter will update the view by invoking the corresponding view method. Usually the view is dump and the presenter commands what the view should display. For example, if the presenter starts a http call to load data the presenter says to the `view.showLoading()`. After the data has been received successfully the presenter would say `view.showData(Data)` or in case of errors `view.showError(Exception)`. A workflow could look like this:

![Model-View-Presenter](/images/mvp_workflow.png)

# MVP on Android

Software architecture on Android is not an easy topic.  Unlikely on iOS where you are forced to use MVC in your whole iPhone application no guidelines or best practice tipps exist in the official android documentation. There are basically two questions: 

**What is a View and what is a Controller on android?** 

Activity and Fragment seems to be both, because they have lifecycle callbacks like `onCreate()` or `onDestroy()` as well as responsiblities of View things like switching from one UI widget to another UI widget like showing a progressbar while loading and then displaying a listview with data. You may say that these sounds like an Activity or Fragment is a Controller and I guess that was the original intention. However after some years in developing Android Apps I came to the conclusion that Activity and Fragement should be treated like a (dump) View and a new component should take the Controller part. Let me explain how I came to this conclusion on a real world app scenario: a simple inbox in an E-Mail client.

The inbox screen (implemented as activity) should look like this: While loading the list of mails a progressbar should be visible on screen. Once the list of emails is loaded it should be displayed on screen instead of the Progressbar. A TextView should be displayed if an error occurres while loading the data to display a error message (instead of showing the progressbar or ListView). No big deal. Basically the inbox screen (View) should provide an API with 3 methods:

 - `showLoading()`: displays a ProgressBar instead of ListView / ErrorView.
 - `showContent()`: displays a ListView with emails instad of ProgressBar or ErrorView.
 - `showError()`: displays a TextView if an error (Excpetion) has occurred instead of ProgressBar or ListView.

Next we want to add Pull-To-Refresh for instance by using `SwipeRefreshLayout` form the support library. Furthermore we want to add fade animations instead of simply changing the visibility of ListView, ProgressView, TextView (error message) from _View.VISIBLE_ to _View.GONE_. We also add a simple database connection that will be queried async triggered by an MenuItem from the ActionBar. While quering the database the ProgressBar should be displayed instead of the ListView. You may notice that the lines of code we have written so far for the activity class grows very fast and may exceeds 1000 lines of code and we haven't even thought about other things like screen orientation changes yet. Do you really think that you can read your own code you have written in two months when you want to add a new feature or fix some bugs? Do you really think one of your co-workers can read and understand your code, especially the parts where showLoading(), showError() and showContent() will be called? (Remember, that there are two components that will trigger this calls: The component that loads the E-Mails and the database component)

In short: your code is a mess because many things are handled (and coded) in the activity class. Wouldn't it be nice if we could strip out code into a separeate component to get clean code, modular components and clear responsiblities?

That's the point where Model-View-Presenter will help you. We start by defining a java interface for our view. We specify in this API what our view can do. Note that for reasons of simplicity we ignore the requirement of heaving a database connection in the following demo code.

{% highlight java %}
public interface InboxView {
  
  public void showLoading(boolean pullToRefresh);
  
  public void showError(Exception e, boolean pullToRefresh);
  
  public void showContent();
  
  public void setData(List<Email> emails);
  
}
{% endhighlight %}

The model (business logic or service or call it whatever you like) is still the same as before:

{% highlight java %}
public class EmailManager {

  public interface EmailFetcherListener{
    public void onSuccessful(List<Email> emails);
    public void onError(Exception e);
  }


  public void getEmailsAsync(EmailFetcherListener listener){
      // TODO fetch the list of emails (async)
  }

}
{% endhighlight %}

Next we refactor the code that controls our screen (displaying ListView, ProgressBar and TextView (error message) by moving that parts from the Activity into the **Presenter**

{% highlight java %}
public class InboxPresenter {

  InboxView view;
  EmailManager emailManager;

  public InboxPresenter(InboxView inboxView){
    this.view = inboxView;
    emailManager = new EmailManager();
  }

  public void loadEmails(final boolean pullToRefresh){

    view.showLoading(pullToRefresh);

    emailManager.getEmailsAsync(new EmailManager.EmailFetcherListener() {
      @Override public void onSuccessful(List<Email> emails) {
        view.setData(emails);
        view.showContent();
      }

      @Override public void onError(Exception e) {
        view.showError(e, pullToRefresh);
      }
    });

  }

  public void onDestroy(){
    // TODO Cancel async running tasks
  }
}
{% endhighlight %}

Do you remember the provocative hypothesis that the **Activity can be seen as a View.** So let's see how the refactored InboxActivity looks like:

{% highlight java %}
public class InboxActivity extends Activity
    implements InboxView, SwipeRefreshLayout.OnRefreshListener {

  @InjectView(R.id.contentView) 
  SwipeRefreshLayout contentView;

  @InjectView(R.id.listView) 
  ListView listView;

  @InjectView(R.id.errorView) 
  TextView errorView;

  @InjectView(R.id.loadingView) 
  ProgressBar loadingView;

  InboxAdapter adapter;

  InboxPresenter presenter;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_inbox);

    ButterKnife.inject(this);
    adapter = new InboxAdapter();
    listView.setAdapter(adapter);
    contentView.setOnRefreshListener(this);

    presenter = new InboxPresenter(this);
    loadEmails(false);
  }

  private void loadEmails(boolean pullToRefresh) {
    presenter.loadEmails(pullToRefresh);
  }

  @Override 
  public void onRefresh() {
    loadEmails(true);
  }

  @Override 
  public void showLoading(boolean pullToRefresh) {

    if (pullToRefresh) {
      contentView.setRefreshing(true);
    } else {

      // TODO animations
      contentView.setVisibility(View.GONE);
      errorView.setVisibility(View.GONE);
      loadingView.setVisibility(View.VISIBLE);
    }
  }

  @Override 
  public void showError(Exception e, boolean pullToRefresh) {

    if (pullToRefresh) {
      Toast.makeText(this, "Error: " + e.getMessage(), Toast.LENGTH_SHORT)
           .show();
    } else {
      // TODO animations
      contentView.setVisibility(View.GONE);
      loadingView.setVisibility(View.GONE);
      errorView.setVisibility(View.VISIBLE);
    }
  }

  @Override 
  public void showContent() {
    // TODO animations
    contentView.setRefreshing(false);
    errorView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    contentView.setVisibility(View.VISIBLE);
  }

  @Override 
  public void setData(List<Email> emails) {
    adapter.setEmails(emails);
    adapter.notifyDatasetChanged();
  }
  
  @Override
  public void onDestroy() {
    super.onDestroy();
    presenter.onDestroy();
  }
}
{% endhighlight %}

Note that the code above should just give you an idea of how to apply the Model-View-Presenter pattern on android. It should not be copy & pasted because there are some issues we will discuss later.

The main idea is that the Presenter is now responsible to control the view and to be the middleman between view and business logic (model). Also the android lifecycle responsiblities has been moved from Activity to the Presenter since the presenter is now responsible to communicate with the business logic. Hence in `Presenter.onDestroy()` the Presenter would cancel any ongoing async tasks, do memory clean up, etc. 

The view has a well defined API that will be implemented by the Activity and hence the actitiviy is a view. You may ask yourself _do I really need to define a interface?_ I strongly recommend that because it brings some advantages:

 1. Since it's an Interface you can change the view implementation. You can simple move your code from something that extends activity to something that extends Fragment.
 2. Modularity: You can move the whole Business logic, Presenter and View Interface in a standalone android library project. Then you can use this library with the containing Presenter in various apps. For instance if you have plans to publish two E-Mail client Apps with different UI you can reuse the same presenter and business logic from your standalone library. So in both _E-Mail App 1_ and _E-Mail App 2_ you only have to impelemt the UI the activity and can reuse everything else from the library.
 3. You can write unit tests since you can make the unit test case implement the view interface. One could also provide a java interface for the presenter unit testing by using mock objects more easy.
 4. Another very nice side effect of defining a interface for the view is that you don't get tempted to call methods of the activity directly. You get a clear separation because while implementing the presenter the only methods you see in your IDE's auto complete are those of the interface. From my personal experiences I can say that this is very useful especially if you work in a team.