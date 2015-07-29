---
layout: post
published: true
title: Joe's great adapter hell escape
mathjax: false
featured: false
comments: true
headline: Joe's great adapter hell escape
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

Let me tell you a story about Joe an android developer at MyLittleZoo Inc. and how he walked through the hell while trying to create reusable RecyclerView Adapters with different view types and how he finally managed to implement reusable Adapters painlessly.

Once upon a time Joe, an android developer, was working for a young startup called MyLittleZoo Inc. This startup was selling stuff for pets online. Joe's job was to build and maintain a native android app which basically offers the same functionality as the online shop (website). So 90% of the android app he had to develop just display a list of items in a `RecyclerView`. The first version 1.0 should just display a list of `Accessories`. Joe implemented an `AccessoiresAdapter` which displays a list of accessories, but special offers for accessories are displayed by using `item_accessory_offer.xml` while the `item_accessory.xml` is used to display any normal accessories item. So the Adapter has two view types. A view type allows you to inflate different xml layouts for different items in the adapter. Internally a view type is just a unique id, an integer. So Joe's `AccessoiresAdapter` implementation looks like this:

{% highlight java %}
public class AccessoiresAdapter extends RecyclerView.Adapter {

  final int VIEW_TYPE_ACCESSORY = 0;
  final int VIEW_TYPE_ACCESSORY_SPECIAL_OFFER = 1;

  List<Accessory> items;

  @Override public int getItemViewType(int position) {
     Accessory accessory = items.get(postion);
     if (accessory.hasSpecialOffer()){
       return VIEW_TYPE_ACCESSORY_SPECIAL_OFFER;
     } else {
       return VIEW_TYPE_ACCESSORY;
     }
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (VIEW_TYPE_ACCESSORY_SPECIAL_OFFER == viewType){
      return new SpecialOfferAccessoryViewHolder(inflater.inflate(R.layout.item_accessory_offer, parent));
    } else {
      return new AccessoryViewHolder (inflater.inflate(R.layout.item_accessory)):
    }
  }

  ...

}
{% endhighlight %}

So far so good, MyLittelZoo android app 1.0 was published on Play Store. Everything was cool.

Then MyLittelZoo grew, so did the app. Joe had to implement a new starting Activity where different items could be displayed: `NewsTeaser` should now be displayed together with `Accessories`. Since `HomeAdapter` should display `Accessories` as well he decided to reuse `AccessoriesAdapter` by inheriting from that one:

{% highlight java %}
public class HomeAdapter extends AccessoriesAdapter {

  final int VIEW_TYP_NEWS_TEASER = 2;

  @Override public int getItemViewType(int position) {
     if (items.get(position) instanceof NewsTeaser){
       return VIEW_TYP_NEWS_TEASER;
     } else {
       // accessories and special offers
       return super.getItemViewType(position);
     }
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (VIEW_TYP_NEWS_TEASER == viewType){
      return new NewsTeaserItem( inflater.inflate(R.layout.item_news_teaser, parent));
    } else {
      // accessories and special offers
      return super.onCreateViewHolder(parent, viewType);
    }
  }

  ...
}
{% endhighlight %}

Also a new Activity just displaying some short tips about pet food should be implemented. Hence Joe implemented `PetFoodTipAdapter`:
{% highlight java %}
public class PetFoodTipAdapter extends RecyclerView.Adapter {

  final int VIEW_TYP_FOOD_TIP = 0;

  @Override public int getItemViewType(int position) {
     return VIEW_TYP_FOOD_TIP;
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return new PetFoodViewHolder(inflater.inflate(R.layout.item_pet_food, parent))
  }

  ...

}
{% endhighlight %}

His project manager was happy since he was able to deliver in time. MyLittelZoo 2.0 was released on Play Store successfully.

A few weeks later product manager came to Joe and told him that business hadn't developed as expected. To earn money the company decided to sign a contract with a big advertisement company. The advertisement company could display banners in MyLittleZoo android app. In other words: they sold their soul to the devil. Joe's job was to include advertisement banner in the app by using a provided advertisement sdk. The clock ticked, the company needed money (revenue from advertisement). The app update had to be published as soon as possible. Since advertisement banner should be displayed along with other items in a RecyclerView Joe decided to create a new base adapter class called `AdvertismentAdapter`:

{% highlight java %}
public class AdvertismentAdapter extends RecyclerView.Adapter {

  final int VIEW_TYP_ADVERTISEMENT = 0;

  @Override public int getItemViewType(int position) {
     return VIEW_TYP_ADVERTISEMENT;
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return new AdvertismentViewHolder(inflater.inflate(R.layout.item_advertisment, parent))
  }

  ...

}
{% endhighlight %}

From then on, every other Adapter extends from `AdvertisementAdapter`:

 - `AccessoiresAdapter extends AdvertisementAdapter`
 - `HomeAdapter extends AccessoiresAdapter extends AdvertisementAdapter`
 - `PetFoodTipAdapter extends AdvertisementAdapter`

Version 3.0 was published on Play Store with advertisement banner everywhere. Once more product manager was happy with Joe's work.

A half year later again the product manager knocked on Joe's door to tell him that things had changed. Surprisingly it turned out that the user of MyLittleZoo android app didn't liked the blinking advertisement banner introduced in version 3.0 and the app got huge negative reviews on play store. User sessions dropped dramatically and company didn't earn money anymore. But MyLittleZoo couldn't simply remove advertisement from the app since they have signed a valid longterm contract with the devil, ehm I mean the advertisement company of course.

Then the smart marketing guy at MyLittleZoo had the brilliant idea to launch a second app with just displaying `NewsTeaser` and `PedFoodTip` in a RecyclerView. No advertisement, no offers. The plan was to regain the confidence of the users. Furthermore, product manager told Joe that the app had to be published within next two days because at the upcoming weekend was a big pet fair where the app should be presented. Joe thought it's doable. He already had the xml layouts for `NewsTeaser` and `PedFoodTip` and the adapter were already implemented. So all Joe had to do is to move that into an android library to share them between original MyLittleZoo app and the new advertisement free app.

Then Joe was about to start moving things into the library when he realized the mess he was facing: Do you remember the inheritance hierarchy of the adapters?

 1. Every Adapter extends from `AdvertisementAdapter`. But no advertisement should be displayed in the new app. Moreover, the provided advertisement sdk to display banners is really buggy, causes memory leaks and crashes quite often. Even if no advertisement banner was displayed the advertisement sdk did a lot of crap in the background. Therefore, including the advertisement sdk in the new app was not acceptable.
 2. There is no adapter that he can reuse that can display `NewsTeaser` (part of `HomeAdapter`) and `PetFoodTip` (part of `PetFoodTipAdapter`). What should Joe do? He could create a new Adapter called `NewsTipAdapter` that  extends from `HomeAdapter` and then he would had to add the `PedFoodTip` as new view type. But that would mean that he would had two Adapters to maintain for the same view type `PedFoodTip`.

 > Welcome to the adapter hell Joe!

Oh boy, Joe was depressed. Then panic followed depression. How should he fix that? How should he fix that without having to fix it again a month later when a new feature (a new view type) must be implemented?

So Joe started to draw on a whiteboard his requirements. But not a single good idea came out. He was so sad, he thought back to the days when he was a little child. How easy was life during childhood. The only thing he had to worry those days was to clean up his room after he had finished to play Lego. Lego? Wait, wait, wait! Joe had a brilliant idea: What he really needs is to built adapter like building a Lego house: Take an empty fundament and then stick together that Lego pieces you really need. If you need a window in your Lego house, take a window piece. If you need a roof slope take the corresponding Lego piece. If your Lego house needs a backyard, take a Lego flower.

Damn, and then he get the overall picture:

 > Favor composition over inheritance

So many times he had agreed on "Favor composition over inheritance" while discussing with other developers. Until now it was just a good slogan but he never really have build something according this principle. So an empty Adapter is the fundament. ViewTypes are the reusable components(Lego pieces).

So Joe started to define reusable Lego pieces called `NewsTeaserAdapterDelegate` and `PedFoodTipAdapterDelegate`:

{% highlight java %}
public class NewsTeaserAdapterDelegate {

  private int viewType;

  public NewsTeaserAdapterDelegate(int viewType){
    this.viewType = viewType;
  }

  public int getViewType(){
    return viewType;
  }

  public boolean isForViewType(List items, int position) {
    return  items.get(position) instanceof NewsTeaser;
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent) {
    return new NewsTeaserViewHolder(inflater.inflate(R.layout.item_news_teaser, parent, false));
  }

  public void onBindViewHolder(List items, int position, RecyclerView.ViewHolder holder) {
      NewsTeaser teaser = (NewsTeaser) items.get(position);
      NewsTeaserViewHolder vh = (NewsTeaserViewHolder) vh;

      vh.title.setText(teaser.getTitle());
      vh.text.setText(teaser.getText());
  }
}
{% endhighlight %}

{% highlight java %}
public class PedFoodTipAdapterDelegate {

  private int viewType;

  public NewsTeaserAdapterDelegate(int viewType){
    this.viewType = viewType;
  }

  public int getViewType(){
    return viewType;
  }

  public boolean isForViewType(List items, int position) {
    return  items.get(position) instanceof PedFoodTip;
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent) {
    return new PedFoodTipViewHolder(inflater.inflate(R.layout.item_pet_food, parent, false));
  }

  public void onBindViewHolder(List items, int position, RecyclerView.ViewHolder holder) {
      PedFoodTip tip = (PedFoodTip) items.get(position);
      PedFoodTipViewHolder vh = (NewsTeaserViewHolder) vh;

      vh.image.setImageRes(tip.getImage());
      vh.text.setText(tip.getText());
  }
}
{% endhighlight %}

And then he took the fundament, an empty adapter, and put the Lego pieces on top of it to create the `NewsTipAdapter` which will be used in the new app:

{% highlight java %}
public class NewsTipAdapter extends RecyclerView.Adapter{

  final int VIEW_TYP_NEWS_TEASER = 0;
  final int VIEW_TYP_FOOD_TIP = 1;

  NewsTeaserAdapterDelegate newsTeaserDelegate;
  PedFoodTipAdapterDelegate foodTipDelegate;

  List items;

  public NewsTipAdapter(){
    newsTeaserDelegate = new NewsTeaserAdapterDelegate(VIEW_TYP_NEWS_TEASER);
    foodTipDelegate = new PedFoodTipAdapterDelegate(VIEW_TYP_FOOD_TIP);
  }

  @Override public int getItemViewType(int position) {
     if (newsTeaserDelegate.isForViewType(items, position)){
       return newsTeaserDelegate.getViewType();
     }
     else if (foodTipDelegate.isForViewType(items, position)){
       return foodTipDelegate.getViewType();
     }

     throw new IllegalArgumentException("No delegate found");
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

    if (newsTeaserDelegate.getViewType() == viewType){
      return newsTeaserDelegate.onCreateViewHolder(parent);
    }
    else if (foodTipDelegate.getViewType() == viewType){
      return foodTipDelegate.getViewType();
    }

    throw new IllegalArgumentException("No delegate found");
  }


  @Override public void onBindViewHolder(VH holder, int position){
    int viewType = holder.getViewType();
    if (newsTeaserDelegate.getViewType() == viewType){
      newsTeaserDelegate.onBindViewHolder(items, position, holder);
    }
    else if (foodTipDelegate.getViewType == viewType){
      foodTipDelegate.onBindViewHolder(items, position, holder);
    }
  }
}
{% endhighlight %}

I guess you get the point. Instead of inheriting Joe had defined a delegate for each view type. Each delegate was responsible for creating and binding a view holder. As you have noticed in the code snipped above, there is a lot of boilerplate code to write. Joe found a smart plugin solution for that:

{% highlight java %}
/**
 * @param <T> the type of adapters data source i.e. List<Accessory>
 */
public interface AdapterDelegate<T> {

  /**
   * Get the view type integer. Must be unique within every Adapter
   *
   * @return the integer representing the view type
   */
  public int getItemViewType();

  /**
   * Called to determine whether this AdapterDelegate is the responsible for the given data
   * element.
   *
   * @param items The data source of the Adapter
   * @param position The position in the datasource
   * @return true, if this item is responsible,  otherwise false
   */
  public boolean isForViewType(@NonNull T items, int position);

  /**
   * Creates the  {@link RecyclerView.ViewHolder} for the given data source item
   *
   * @param parent The ViewGroup parent of the given datasource
   * @return The new instantiated {@link RecyclerView.ViewHolder}
   */
  @NonNull public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent);

  /**
   * Called to bind the {@link RecyclerView.ViewHolder} to the item of the datas source set
   *
   * @param items The data source
   * @param position The position in the datasource
   * @param holder The {@link RecyclerView.ViewHolder} to bind
   */
  public void onBindViewHolder(@NonNull T items, int position, @NonNull RecyclerView.ViewHolder holder);
}
{% endhighlight %}

{% highlight java %}
public class AdapterDelegatesManager<T> {

  public AdapterDelegatesManager<T> addDelegate(@NonNull AdapterDelegate<T> delegate) {
    ...
  }

  public int getItemViewType(@NonNull T items, int position) {
    ...
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    ...
  }

  public void onBindViewHolder(@NonNull T items, int position, @NonNull RecyclerView.ViewHolder viewHolder) {
    ...
  }
}
{% endhighlight %}

The idea is to register `AdapterDelegates` to an `AdapterDelegatesManager`. The `AdapterDelegatesManager` internally has the logic to determine the correct adapter delegate for the given view type and to call the corresponding delegate method. So applying that to `NewsTipAdapter` the code looks like this:
{% highlight java %}
public class NewsTipAdapter extends RecyclerView.Adapter{

  final int VIEW_TYP_NEWS_TEASER = 0;
  final int VIEW_TYP_FOOD_TIP = 1;

  List items;

  AdapterDelegatesManager delegates = new AdapterDelegatesManager();

  public NewsTipAdapter(){
    delegates.add(new NewsTeaserAdapterDelegate(VIEW_TYP_NEWS_TEASER));
    delegates.add(new PedFoodTipAdapterDelegate(VIEW_TYP_FOOD_TIP));
  }

  @Override public int getItemViewType(int position) {
     return delegates.getItemViewType(items, position);
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return delegates.onCreateViewHolder(parent, viewType);
  }

  @Override public void onBindViewHolder(VH holder, int position){
      delegates.onBindViewHolder(items, position, holder);
  }
}
{% endhighlight %}

I guess you can imagine how other adapters of MyLittleZoo app looks now. There is an `AdvertisementAdapterDelegate`, `NewsTeaserAdapterDelegate`, `PedFoodTipAdapterDelegate` and `AccessoryAdapterDelegate`. From now on adapters can be composed with that view types (AdapterDelegates) that are really needed. Another advantage is that you also have moved out the functionality of inflating layout, creating view holder and binding view holder from one huge big adapter (spagehtti code?) into separated, modular and reusable AdapterDelegates. Have you noticed how slick adapters code looks now and that you have a separation of concerns that makes things more extendable and more decoupled?

Joe was happy, the product manager was happy, the users of the app was happy. Everybody was happy.
Actually, Joe was so happy that he had decided to put `AdapterDelegates` in a own library and open source it.

[You can find AdapterDelegates on Github](https://github.com/sockeqwe/AdapterDelegates) and is available in maven central.

<small>
Please note that the code snippets shown in this blog post may not compile. It's kind of java alike pseudo code to give you an idea of how the code could look like.
</small>
