---
layout: post
published: true
title: Presenters don't need lifecycle events
mathjax: false
featured: false
comments: true
headline: Presenters don't need lifecycle events
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

I have been asked several times why Presenters in [Mosby](https://github.com/sockeqwe/mosby) (MVP library) don't have lifecycle callback methods like onCreate(Bundle), onResume() etc. Also the awesome guys over at SoundCloud have published a library called [LightCycle](https://github.com/soundcloud/lightcycle) that helps break logic out of Activity or Fragments into smaller containers bound to the parents Activity's or Fragment's lifecycle. While this library is great and helpful they also mention in their [examples](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java) that this library can be used in MVP to bring lifecycle to Presenters. I, personally, think that Presenters don't need lifecycle callback methods and in this blog post I will discuss why.

Alright, before we get started, think of the definition and the rule of a `Presenter` in MVP. Here is my definition:

> The Presenter is responsible to coordinate the view, transform Model into "PresentationModel" so that the View can display data more convenient and to be the bridge to the "business logic" to retrieve data that should be displayed in the View. MVP is all about separation of concerns.

Given that "definition" of a Presenter I don't see a reason why the Presenter needs lifecycle callbacks like `onCreate()`, `onPause()` and `onResume()`. All the Presenter needs to know is whether or not a View is attached to the Presenter (because of androids screen orientation dilemma). It should be that simple. Actually, this is one of the arguments why some developers are advocating against fragments (complex lifecycle) and prefer "custom views" (i.e. extending from FrameLayout) because custom view's only have two lifecycle events: ViewGroup.onAttachedToWindow() and ViewGroup.onDetachedFromWindow() (btw. that's the reason why [Mosby Presenters](https://github.com/sockeqwe/mosby/blob/master/mvp-common/src/main/java/com/hannesdorfmann/mosby/mvp/MvpPresenter.java) only have `attachView()` and `detachView()` methods if you want to count them as "lifecycle events").

Furthermore, lifecycle managing is already a complex topic. If you start to move lifecycle logic in your Presenter then you basically have just moved that spaghetti code from Activity to Presenter. Clearly the intention of [LightCycle](https://github.com/soundcloud/lightcycle) is to solve exactly this problem: LightCycle helps you to split or delegate the spaghetti code logic you usually write in your Activity's or Fragment's lifecycle methods into multiple smaller components that are following the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle). However, according to LightCycle [examples](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java) they promote (indirectly) to build Presenters with lifecycle methods. Don't get me wrong: The idea of LightCycle is great and achieving that via annotation processing is pretty smart, but lifecycle management is not the Presenters responsibility and therefore the Presenter doesn't need lifecycle callback methods. This blog post is not advocating against LightCycle (it's awesome!). This blog post is advocating against lifecycle in Presenters in general!

With that said, you might understand my point of view, but you also have faced scenarios where from your point of view the Presenter indeed needs lifecycle callback methods. Okay, let's talk about it!

For example let's assume we want to build an app that displays the user's current GPS position on a map in our app. As the screen turns off we should stop retrieving GPS position updates to safe battery life. So how would we implement this with MVP? We would have an `TrackingActivity` as View (`TrackingView`) displaying a map with the user's current GPS position. We also need a `GpsTracker` class as business logic" (model in MVP) which is responsible to detect the current GPS position and notify observers (listeners) about position changes. The `TrackingPresenter` is such an observer. He is registered as listener to `GpsTracker` and will update the View when GPS position has been changed.

So far so good. As already said, to not drain the battery too much we want to stop GPS tracking when the display is off, in other words stop GPS Tracking in `Activity.onPause()` and resume when `Activity.onResume()`. Read this sentence carefully again. Do you get it? It's not the Presenter who needs onPause() and onResume() lifecycle events. Rather the "business logic", the `GpsTracker`, needs these lifecycle events.

But how do we implement that? Should we simply forward `Activity.onPause()` to `Presenter.onPause()` which then calls `GpsTracker.stop()`? So we do need a Presenter with lifecycle methods otherwise we couldn't forward the pause events the way down to `GpsTracker.stop()`, right?
I think there is a better way. As already said, is not the responsibility of the Presenter to handle lifecycle events. Actually, there is already a component that is responsible for lifecycle events: the Activity (or Fragment). So instead of forwarding Activity.onPause() and Activity.onResume() to the presenter, just do something like this:

```java
class TrackingActivity extends MvpActivity implements TrackingView {
     private GpsTracker tracker;

     public onCreate(){
            tracker = new GpsTracker(this); // might need a context
            ...
     }

     public void onPause(){
         tracker.stop();
     }

    public void onResume(){
         tracker.start();
     }

    @Override
    public void createPresenter(){ // Called by Mosby
          return new TrackingPresenter(tracker);
     }
}
```

As you see, the trick is that the `GpsTracker` is "bound" to the Activity's lifecycle directly in the component that is responsible for managing lifecycle: the Activity! Then we pass the GpsTracker to the Presenter as constructor parameter.
Furthermore, now `TrackingPresenter` fulfills the single responsibility from my previous definition: It's only responsible to update the View.

```java
class TrackingPresenter extends MvpBasePresenter<TrackingView>
                        implements GpsUpdateListener{

  GpsTracker tracker;

  public TrackingPresenter(GpsTracker tracker){
    this.tracker = tracker;
    tracker.setGpsUpdateListener(this);
  }


  @Override
  public void onGpsLocationUpdated(GpsPosition position){
    view.showCurrentPosition(position.getLat(), position.getLng());
  }
}
```

That's it. **Single responsibility for all your components!**

Of course we can spice up the code shown above with dependency injection and with LightCycle:

```java
class TrackingActivity extends MvpActivity implements TrackingView {

    @Inject @LightCycle GpsTracker tracker;

    @Override
    public TrackingPresenter createPresenter(){
          return getObjectGraph.get(TrackingPresenter.class); // Dagger 1
          // or
          return getComponent().trackingPresenter(); // Dagger 2
     }
}
```

Are you looking for more examples? Another user of Mosby (MVP library) has asked me a similar question on github for a music player app he is working on. I gave a similar  [answer](https://github.com/sockeqwe/mosby/issues/124).

Of course this is only my personal opinion and there are exceptions and you may really need lifecycle events in your Presenters, but I think in 99% your business logic needs lifecycle events and not your Presenters. Usually (I would say 95% of the apps out there) don't have lifecycle aware business logic. And if an app really has such lifecycle aware business logic, then 99% of the time the Presenter doesn't have to be lifecycle aware, but rather the business logic.

So the moral of the story is: You should avoid lifecycle aware components as much as possible. Doesn't matter if business logic (Model) or Presenter. It's my general advise. Lifecycle only introduces undesired complexity. But if you really really really need components that are lifecycle aware, then most likely the business logic (Model) should be lifecycle aware and not the Presenter.

Last but not least I want to say that the intention of this blog post is not to discredit SoundCloud's LigthCycle! I just wanted to say that from my point of view the LightCycle [examples](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java) (having Presenters with lifecycle callback methods in general) are not my cup of tea. I think this is mainly caused by the fact that my definition of a Presenter (i.e. I also don't want to have android SDK dependencies like `Bundle` in my Presenters) is entirely different from SoundClouds definition of a Presenter.

### Update
As some people pointed out correctly (see also [reddit](https://www.reddit.com/r/androiddev/comments/4bs4lz/presenters_dont_need_lifecycle_events_hannes/) ) the code snipped shown above causes that the `TrackingView` (TrackingActivity) has knowledge of `GpsTracker` which definitely feels dangerous. What I wanted to demonstrate is that there is already a component responsible for lifecycle management in your app and it's not the Presenter. I didn't say that `GpsTracker` should be used or manipulated from `TrackingView` directly. That should still be the job of the `TrackingPresenter`. However, yes, `TrackingActivity` now has a reference to `GpsTracker` and could be misused. The real problem is that `Activty` is `View` and `lifecycle manager` at the same time. I see two solutions for that problem:

1. **Separate View and lifecycle responsibility:** How do we do that? Well we have to move either View responsibility or lifecycle management out of Activity. Obviously we can't remove the lifecycle management from Activity easily, but we can introduce an additional layer which then is really just a `TrackingView`:

 ```java
class TrackingActivity extends MvpActivity<TrackignView, TrackingPresenter> {

   @Inject @LightCycle GpsTracker tracker;
   TrackingView view;

   public void onCreate(n){
     super.onCreate(b);
     setContentView(R.layout.activity_tracking);
     view = (TrackingView) findViewById(R.id.tracking_view);
   }

   @Override // called by Mosby, same as before
   public TrackingPresenter createPresenter(){ ... }

   @Override // called by Mosby to connect view with presenter
   public TrackingView getMvpView(){
     return view;
   }
}
 ```

 The thing here is we introduce an extra layer View. This can be something like `TrackingLayout extends FrameLayout implements TrackingView` so you can use it directly in xml layouts or you could build some kind of "wrapper class" that gets the root layout from Activity and internally manages UI widgets or whatever works best for you. This approach might be the cleanest solution because now we have truly single responsibility even in the Activity (lifecycle only). But this solution comes with a price: an additional layer. I don't know about you but for me adding yet another layer is the beginning of over-engineering. And there are still some dependencies an activity has to offer to the view layer like what if you need a Bundle in View (do you really want to work with BaseSavedState)? What if the view needs a reference to the Window to set some flags like hide navigation bar, immersive mode or shared element transitions. Maybe we could use dependency injection like Dagger to solve that kind of problems? Maybe, but soon or later you will be caught by dependency injection library so that we can't do anything without dagger anymore, which sucks. So even if in theory introducing a dedicated "view" layer seems to be a good idea it's not my recommended solution.

 2. **Hide GpsTracker from View:** This seems a like a work around, but from my experience this is the simpler and a more practical solution. What was the original problem? The problem was that `TrackingActivity` has a reference to `GpsTracker` and therefore one could access and misuse `GpsTracker`. How to solve this problem? Introducing an additional layer seems to me like getting the fire brigade to extinguish a candle. So why not simply "hide" `GpsTracker` from `TrackingActivity` like this:

 ```java
class LifecycleController  extends DefaultActivityLightCycle {
  private GpsTracker tracker;

  @Inject
  public LifecycleController(GpsTracker tracker){ ... }

  @Override
  public void onPause(){
    tracker.stop();
  }

  @Override
  public void onResume(){
    tracker.start();
  }
}
 ```

 and then we tread the Activity as `TrackingView` as we already did before:

 ```java
class TrackingActivity extends MvpActivity implements TrackingView {

    @Inject @LightCycle LifecycleController lifecycleController;

    @Override
    public void createPresenter(){ // Called by Mosby
          return new TrackingPresenter(tracker);
     }
}
 ```

 By doing so `TrackingActivity` no longer has a reference to `GpsTracker` that can be misused without the burden of an additional layer.

Please note, that in this blog post we are talking about business logic components like `GpsTracker` that are lifecycle aware. Obviously, I don't want you to do that for all your business logic components even if they are not lifecycle aware at all. That is nonsense. The "traditional" MVP approach is quite good. I was just advocating against making Presenter lifecycle aware when the business logic is the component that is lifecycle aware.

<small>I apologize for having inserted this solution discussed in the "Update" section after having published the blog posts. Initially I was too lazy to formalize all that things out. I'm sorry.</small>
