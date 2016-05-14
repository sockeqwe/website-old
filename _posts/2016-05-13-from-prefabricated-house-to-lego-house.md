---
layout: post
published: true
title: From Prefab House to Lego House
mathjax: false
featured: false
comments: true
headline: From Prefabricated House to Lego House
categories:
  - Android
tags: [android, software-architecture]
imagefeature: legohouse/legohouse.jpg
---

Before joining my current employer I have worked for an app agency and one thing I have noticed is that we were building every app entirely new from scratch even if we were building a similar app for the same customer with similar UI. We hadn't reusable components. In this blog post I want to share some thoughts and lessons learned while building such reusable components.

At my current employer, [Tickaroo](https://www.tickaroo.com), we were facing the same situation three years ago. Although Tickaroo is not an app agency, it's a startup for sport enthusiasts, we are building apps for our partners from the world of sports as well. One of this partner is the _Olympia Verlag_ , a renowned publisher in Germany for which we have started building (and are still developing and maintaining) their native android and iOS apps for one of their magazines called [kicker](https://play.google.com/store/apps/details?id=com.netbiscuits.kicker) (football magazine, which displays football news, live updated and so on) three years ago.

After having released the first versions of the [kicker](https://play.google.com/store/apps/details?id=com.netbiscuits.kicker) app successfully in 2013 the _Olympia Verlag_ was pleased with our work and charged us with building another app called  [MeinVerein](https://play.google.com/store/apps/details?id=com.tickaroo.meinverein), also a football app where you can select your favorite football club and then the app displays news and games about your football club.

While kicker app is quite a huge app that can be used for free (with advertisement), MeinVerein is a paid only app (subscription based). MeinVerein's app functionality can be seen as some kind of subset from kicker app.

One thing was clear from the very beginning: since the same designer who already had designed the kicker app was going to design MeinVerein app too the UI should be quite similar for both apps. In fact, this was desired by our customer _Olympia Verlag_ since they wanted to have a consistent design branding across all of their apps. Let's compare the UI of the apps:

![UI comparison](/images/legohouse/comparison.png)

Basically MeinVerein app displays different data in more or less the same way as kicker app does with just some other colors / theme and the main navigation of both apps are different (navigation drawer vs. ViewPager). Some screens, for example the live updates screens, are completely identical. Other screens available in kicker app are not available in MeinVerein app. Additionally, MeinVerein offers some infos and screens kicker doesn't. Also, even if the UI seems pretty similar completely different actions and screens will be opened by for example clicking on a button. So MeinVerein should provide the same look and feel as kicker app, but you can't say that it is a simple build flavor / variant of the kicker app.

## Let's build a prefabricated house
Before we started with the development of MeinVerein app we, the iOS team and the android team, get together and discussed how we could build that app without much effort and in little time. The leading iOS developer, which by the way is the most experienced developer in our company, suggested to put all the stuff from the kicker app into a library. Software development is much about experience. Hence, it is priceless to have an experienced developers in your team and we from the android team had zero experience on how to build such a shared code base. He suggested to build a shared library as fundament for both, kicker app and MeinVerein app. So we did. The kicker app was build by using Fragments. Basically every screen is a Fragment, NewsListFragment to display a list of news, LeagueTableFragment etc. Furthermore, we have built this app by using [Mosby](http://hannesdorfmann.com/mosby/) a Model-View-Presenter library and so we moved all Fragments (MVP View), RecyclerView Adapters, XML layouts, Presenters and Model classes from kicker app into the newly created shared library. kicker app's git repository was quite empty afterwards. For the sack of simplicity we only will discuss things in this blog post by using NewsListFragment as representative as a screen that looks similar in both apps, but keep in mind that there are also many other Fragments in this shared library.

The vision was to build screens for the corresponding apps by extending from the corresponding Fragment of the shared library that has already his Presenter, his RecyclerView Adapter, his XML layouts and so on. Customization should be done by overriding some methods from shared library's Fragment, coloring and theming in xml and if required provide a custom dagger module to inject non default app specific components via dependency injection. The iOS team did the same with their UIViewControllers etc. In a nutshell: We built a prefabricated house where we can specify the color of the walls.

## The problems of our prefabricated house
Thanks to the shared library we were able to ship MeinVerein app within a few weeks. Everything was good. But we noticed the first symptoms: We caught ourselves programming 90% of the time in the shared library project even though that the actually code we were writing was kicker app exclusively and not even used by MeinVerein. But we ignored those symptoms: We use proguard which will remove all the unused classes from shared library in MeinVerein app or kicker app respectively, right?

Another example: We decided to put all our model classes and http networking stuff in the library as well, because even if the backend is not the same (different url) the backend guys said to us that the json data feed structure will be the same. At least that was the case in the first version of MeinVerein but as time went by and more features had been implemented the data feeds for both backends begin to differ more and more. However, imho the real problem was that all the pitfalls and issues on kicker backend side regarding json feed structure had also been copied to MeinVerein backend instead of avoiding this issues from very beginning since MeinVerein backend was a greenfield. But you know, we had to move fast, so had the backend guys. Today I know: you don't move fast by writing bad code.

Overall our shared library looks like this, i.e. to load and display a list of news items (NewsListFragment):

![NewsListFragment](/images/legohouse/NewsListFragment.png)

Basically we used `NewsListFragment` directly in kicker app and `MVNewsListFragment extends NewsListFragment` in MeinVerein app with some overridden methods to achieve our goals. Looks reasonable, doesn't it?

Unfortunately, the devil is in the detail. While kicker app has advertisement, MeinVerein app has not advertisement because it's a paid app. Also some features available in kicker App are not available MeinVerein app.

![No advertisement](/images/legohouse/no-advertisement-feature.png)

Those are just two of many UI differences between both apps that has been envoled over the time.
Sure, it's our fault because the inheritance hierarchy is incorrect: The shared library shouldn't provide a NewsListFragment with advertisement and feature XY build in but there should be rather a `AdvertismentNewsListFragment extends NewsListFragment` in the kicker app only, right?
A few months later we started building a new app called "Tippspiel app" also for _Olympia Verlag_ that looks like this:

![Tippspiel](/images/legohouse/kicker-meinverein-tippspiel.png)

Tippspiel looks similar to kicker and MeinVerein app. Actually, is uses the same `NewsListFragment`. However, apart from displaying a list of news in the same way as kicker and MeinVerein this app has nothing in common with the other apps. Tippspiel app has advertisement in the NewsList but not the feature XY. So what actually should `NewsListFragment` provide? Clearly we should introduce a inheritance hierarchy like this:

![Tippspiel](/images/legohouse/inheritance-newslistfragment.png)

So should we put `NewsListFragment `, `AdvertisementNewsListFragment `, `FeatureXYNewsListFragment ` and `AdvertismenetFeatureXYNewsListFragment ` into the shared library so that we can choose the right base class that matches the requirements for the app like MeinVerein `MVNewsListFragment extends NewsListFragment` (no advertisement, no feature XY), kicker `KikNewsListFragment extends AdvertismenetFeatureXYNewsListFragment ` (advertisement and feature XY) and Tippspiel `TPNewsListFragment extends AdvertisementNewsListFragment `(advertiesment only)? You see, inheritance doesn't solve that problem, because for each feature you would have to introduce a new class in your inheriatance hierarchy and there are so many possible combinations for advertisement support, feature A, feature B, feature XY etc. Single inheritance as in java doesn't scale. I have written an blog post [Mixins as alternative to inheritance](http://hannesdorfmann.com/android/java-mixins) that you may find useful in this context.

But wait ... there is more: lately we have worked on the newest member of _Olympia Verlag_ apps called Nordbayern app (another magazine):

![Tippspiel](/images/legohouse/kicker-meinverein-tippspiel-nordbayern.png)

So we decided to keep one single `NewsListFragment` in the shared library that has advertisement support and all features already implemented. Our solution for that problem was to extend from the "super provide everything NewsListFragment" but then inject a feature XY component that does nothing (stub) via dagger if the feature XY is not supported in the app. For Example, MeinVerein App doesn't have feature XY, so we injected a featureXY component that does nothing in MeinVerein app whereas kicker app uses the build in feature XY component that does something useful. We thought about this as some kind of workaround to "remove unused" functionality. That made our shared libraries code confusing and debugging wasn't easy because you didn't know if a invoked method of a feature component does something useful or is just a stub. But we hadn't a better solution.

But wait ... there is more: When we started to build our prefabricated house (a.k.a shared library) we were under the very naive illusion that now we only have to write one unit test for the `NewsListFragment` from shared library once for all apps and also if we would detect a bug in NewsListFragment we would fix it once for all apps. The reality was quite the opposite. Since, every app extended from `NewsListFragment` but has overridden some methods in the concrete apps fragment, we couldn't be sure that the bugfix really fixed the problem in all apps and that the fragment is working as expected. Moreover, with every change or new feature we added to shared library's `NewsListFragment` we had to fear that this will break things in other apps extending from this fragment. Our daily work flow was something like this: Write some code in share library (~ 10 minutes of work), test that each app is still working properly (~ 10 minutes of work for each app thanks to gradle incredible fast build time). So we spent around 1 hour for a code change doable in 10 minutes.

But wait ... there is more: until now we only talked about UI related things. But there are some other stuff like Tracking which are tight to lifecycle, one app has multiple tracking for some strange reason, another app uses an advertisement sdk which is also lifecycle aware and you have to call some methods like `advertismentView.stop()` in `Fragment.onDestroyView()` etc. Put all those things and dependencies into `NewsListFragment` in shared lib? It was a nightmare.


## The need of a Lego kit
We from the android team decided to take on this problem. Things got completely out of hand and we were slowed down by our shared library which actually should help us go fast. To be fair, when we started with the shared library it wasn't predictable that we were going to build so many apps based on this shared library in the future and that requirements and features were that completely different from app to app.

We started to brainstorm and we made an interesting observation: Actually, writing a Presenter is super easy and can be done within a few minutes. So let's remove the Presenters from shared library and let every app have their own Presenters. The same is true for our http library: communicating with a backend with Retrofit just requires to define a service interface with some Retrofit annotations. That takes only a few minutes. Also the "Model" classes like `NewsList` were part of the shared library. Defining your own Model class for each app is not a huge deal and is also not really time intensive. Obviously, those things (Presenter, http and "model" classes) weren't the biggest issue, but removing that from shared library pointed us towards to a general solution: Only keep that things in a shared library that are time intensive to code again and again. We came to the conclusion that the most time consuming thing for us was writing RecyclerView ViewHolders and corresponding XML layout files. Also, time intensive was to implement all the lifecycle aware stuff like tracking and so on and **NOT** implementing a Fragment or Activity class per se.

Once we realized that the solution was obvious: We don't want to build a prefabricated house, we want to build a Lego house where we can only take those Lego pieces we really need for that app. In other words: we needed a **Lego kit** containing ViewHolders and XML layouts and also a plugin mechanism to hook in tracking and advertisement into Fragment's lifecycle.

So we decided to deprecate the shared library and build two libraries: One containing ViewHolders, one containing plugin components for lifecycle. Another thing was very clear to us:

 > Favor composition over inheritance

 We didn't wanted to make the same mistake again: a god alike `NewsListFragment` to extend from.

 First, let's talk about how we decided to deal with `ViewHolder`. Well, there is a library of mine which I have build in this context called [AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates). The idea is simple: defining a delegate for each RecyclerView's ViewType / ViewHolder which does the job of inflating xml layout and binding the data:

```java
 public class BigNewsItemAdapterDelegate extends AbsListItemAdapterDelegate<BigNewsItem, NewsViewHolder> {

     @Override public NewsViewHolder onCreateViewHolder(ViewGroup parent) {
       return new CatViewHolder(inflater.inflate(R.layout.item_cat, parent, false));
     }

     @Override public void onBindViewHolder(BigNewsItem item, NewsViewHolder vh) {
       vh.title.setText(item.getTitle());
       loadImage(vh.image, item.getImageUrl());
     }
}
```

So we defined all those AdapterDelegates like `BigNewsItemAdapterDelegate`,  `VideoNewsItemDelegate`, `SlideshowNewsItemDelegate` etc. and put them in a library.

Now kicker's `KikNewsListFragment` looks like this:

```java
@Override
public void onCreateView(LayoutInflater inflater, ViewGroup parent, Bundle b){
    ...
    RecyclerView rv =  ...;

    Adapter adapter = new DelegatingAdapter<NewsItem>();
    adapter.add(new BigNewsItemAdapterDelegate())
           .add(new VideoNewsItemDelegate())
           .add(new SlideshowNewsItemDelegate())
           .add(new AdvertisementDelegate());

   rv.setAdapter(adapter);
}
```

Please note that the `AdvertisementDelegate()` is not part of the library, because this one is used in kicker app and but not in MeinVerein app. I guess you know how the code of Fragments of the other apps  look like, right? They reuse the same AdapterDelegates, the same xml layouts etc. However, to be really independent every AdapterDelegate has a tiny companion class for holding the required data to be displayed in a ViewHolder as well, i.e. `BigNewsItemAdapterDelegate` has a `BigNewsItem`, `VideoNewsItemDelegate` has a `VideoNewsItem` and so on.

So at some point we have to map the data coming from backend to this companion object. This is where a MVP based App pays of. This is the responsibility of the Presenter. We transform the "model" into a "presentation model". Overall the architecture looks like this:

![Tippspiel](/images/legohouse/adpterdeleages-architecture.png)

And for the Tracking stuff? Favor composition over inheritance!

```java
interface LifecycleDelegate {
  void onCreate();
  void onPause();
  void onResume();
  void onDestroy();
  ...
}

class TrackingLifecycleDelegate implements LifecycleDelegate {
  private String trackingTag;
  public TrackingLifecycleDelegate(String trackingTag){
     this.trackingTag = trackingTag;
  }

  @Override
  public void onResume(){
    Tracking.viewAppeared(trackingTag);
  }
  ...
}

class KikNewsListFragment extends Fragment {

  List<LifecycleDelegate> lifecycleDelegates = new ArrayList<>();

  { // Initializer block
    lifecycleDelegates.add(new TrackingLifecycleDelegate("kicker app NewsListScreen"));
  }

  @Override
  public void onResume(){
    super.onResume();
    for (LifecycleDelegate l : lifecycleDelegates){
      lifecycleDelegates.onResume();
    }
  }

  // Same for other lifecycle methods
  ...
}
```

Now we can share `TrackingLifecycleDelegate` with all apps. Overall it looks like this:

![Tippspiel](/images/legohouse/alldelegates-architecture.png)

There are some libraries you can use to achieve the same thing: Soundcloud's [Lightcycle](https://github.com/soundcloud/lightcycle) and [AndroidComposite](https://github.com/passsy/CompositeAndroid) by  Pascal Welsch.

## Summary
In my opinion a Lego house is much more flexible and maintainable compared with a prefabricated house. Favor composition over inheritance is the key to build a Lego kit. Obviously, there is no silver bullet to solve that kind of problem. Our Lego house approach works quite good for us, but i.e. our  iOS team still prefer the prefabricated house approach even though they have similar issues as we had in our android apps. Somehow that approach works for them (but don't ask me how). I guess the important bit is that all developers working in the same team know exactly how this approach should be implemented and have committed to do that consistently the same way. Maybe we from the android team failed at this point.

Was it an error to start with the prefabricated house approach suggested by the most experienced developer in our company? No! MeinVerein app has been build with that approach in incredible little time. I definitely would follow the experienced developers recommendation again, but not blindly. I have learned a lot, which hopefully will help me with future decisions. What really matters is to learn from the past to build a better future. I think with the Lego house approach we are moving in the right direction. But who knows, maybe in two years I will write a blog post about how bad the Lego house approach was.
