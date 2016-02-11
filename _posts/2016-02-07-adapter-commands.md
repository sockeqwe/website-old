---
layout: post
published: true
title: AdapterCommands
mathjax: false
featured: false
comments: true
headline: AdapterCommands
categories:
  - Android
tags: [android, design-patterns]
---

Last week I was honored to be guest at Artem Zinnatullin's podcast [The Context](https://github.com/artem-zinnatullin/TheContext-Podcast) where we talked about software architecture on android. In this episode I have highlighted how important an presentation model in MVP is by giving an example how to deal with RecyclerView Adapters dataset changes. Afterwards people asked me how exactly do I apply animated dataset changes and why a presentation model is helpful in this case.

Before we dive into the presentation model part, here are the good news: I have put all those things together and bundled it into a library called [AdapterCommands](https://github.com/sockeqwe/AdapterCommands).

So what is this library all about? Well, `RecyclerView` has this nice component called `ItemAnimator` which is responsible to animate items of RecyclerView. For example, let's say we are displaying a list of items in a RecyclerView. When adding a new item we can call `adapter.notifyItemInserted(position)` rather than just `adapter.notifyDatasetChanged()` to specify what exactly has been changed (we have inserted an item). Now `ItemAnimator` kicks in and animates the item in.

`AdapterCommands` basically implements the [command pattern](https://en.wikipedia.org/wiki/Command_pattern) in which a `Command` object is used to encapsulate all information needed to perform an action. So instead of calling `adapter.notifyItemInserted(position)` directly this library provides a `ItemInsertedCommand` which looks like this:

{% highlight java %}
public class ItemInsertedCommand implements AdapterCommand {

  private final int position;

  public ItemInsertedCommand(int position) {
    this.position = position;
  }

  @Override
  public void execute(RecyclerView.Adapter<?> adapter) {
    adapter.notifyItemInserted(position);
  }

}
{% endhighlight %}

As you see this class implements the interface `AdapterCommand` which basically has a `execute(adapter)` method. This library offers such commands for all this actions like `ItemRemovedCommand`, `ItemChangedCommand`, `ItemRemovedCommand` and so on. This library also provides a class `AdapterCommandProcessor` that takes a `List<AdapterCommand>` and executes each command:

{% highlight java %}
public class AdapterCommandProcessor {

  private final RecyclerView.Adapter<?> adapter;

  public AdapterCommandProcessor(RecyclerView.Adapter<?> adapter) {
    this.adapter = adapter;
  }

  public void execute(List<AdapterCommand> commands) {
    for (int i = 0; i < commands.size(); i++) {
      commands.get(i).execute(adapter);
    }
  }
}
{% endhighlight %}

I know, it's not that impressive at first glance. So what is now the advantage of this pattern?
Quite often a app displays a list of items in a RecyclerView and the underlying dataset gets changed, i.e. in combination with `SwipeRefreshLayout` the user can reload an updated list of items (i.e. load items from backend). What do we do with the new list? Just call `adapter.notifyDatasetChanged()` to inform that the new list of items should be displayed? But what about the `ItemAnimator`? This `AdapterCommands` library offers `DiffCommandsCalculator` class. This class calculates the difference of the old list and the new list and returns a `List<AdapterCommand>` that then can be executed by an `AdapterCommandProcessor`. Let's have a look at the demo:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/z05IK8ejERM" frameborder="0" allowfullscreen></iframe>
</p>

As you see in the demo video above, whenever we click on the add or remove button item changes are animated. The implementation looks like this:

{% highlight java %}
public class MainActivity extends AppCompatActivity
    implements SwipeRefreshLayout.OnRefreshListener {

  @Bind(R.id.recyclerView) RecyclerView recyclerView;

  List<Item> items = new ArrayList<Item>();
  Random random = new Random();
  ItemAdapter adapter; // RecyclerView adapter
  AdapterCommandProcessor commandProcessor;
  DiffCommandsCalculator<Item> commandsCalculator = new DiffCommandsCalculator<Item>();

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    ButterKnife.bind(this);

    refreshLayout.setOnRefreshListener(this);

    adapter = new ItemAdapter(this, items);
    recyclerView.setAdapter(adapter);
    recyclerView.setLayoutManager(new GridLayoutManager(this, 4));

    commandProcessor = new AdapterCommandProcessor(adapter);
  }

  @OnClick(R.id.add) public void addClicked() {

    int addCount = random.nextInt(3) + 1;

    for (int i = 0; i < addCount; i++) {
      int position = random.nextInt(items.size());
      Item item = new Item(id(), randomColor());
      items.add(position, item);
    }
    updateAdapter();
  }

  @OnClick(R.id.remove) public void removeClicked() {

    int removeCount = random.nextInt(3) + 1;

    for (int i = 0; i < removeCount; i++) {
      int position = random.nextInt(items.size());
      Item item = items.remove(position);
    }
    updateAdapter();
  }

  private void updateAdapter() {
    // calculate the difference to previous items
    List<AdapterCommand> commands = commandsCalculator.diff(items);
    commandProcessor.execute(commands);
  }

}
{% endhighlight %}

# Presentation Model
I guess you get the point, but how is this related to the presentation model and MVP? In MVP the Presenter (can) generates a `PresentationModel` which is a data model optimized for the view containing all the information the view needs to know so that the view can simply take this presentation model and can display it directly without having to calculate things. More information can be found [here](https://github.com/sockeqwe/mosby/issues/85).

So lets assume we are building an app for a newspaper by applying MVP, using Retrofit to load a list of NewsItems and use RxJava to connect the dots. Instead of passing a `List<NewsItem>` directly from Presenter to View we introduce a `NewsItemsPresentationModel` that looks like this:

{% highlight java %}
class NewsItemsPresentationModel {
  List<NewsItem> newsItems;
  List<AdapterCommand> adapterComamnds;
}
{% endhighlight %}

With RxJava it's quite easy to transform the `List<NewsItem>` to a `NewsItemsPresentationModel` by defining a `Func1` like this:

{% highlight java %}
class PresentationModelTransformer extends Func1< List<NewsItem>, NewsItemsPresentationModel> {

  private DiffCommandsCalculator<NewsItem> diffCalculator = new DiffCommandsCalculator<>();

  @Override
  public NewsItemsPresentationModel call(ist<NewsItem> items){
    List<AdapterCommand> commands = diffCalculator.diff(items);
    return new PresentationModelTransformer(items, commands);
  }
}
{% endhighlight %}

The Presenter looks like this:

{% highlight java %}
class NewsItemsPresenter extends MvpBasePresenter<NewsItemView> {

  private BackendApi backendApi; // Retrofit service to load news items
  private PresentationModelTransformer pmTransformer = new PresentationModelTransformer()

  public void loadItems(){
    view.showLoading();
    backendApi.getNewsItems()
              .map(pmTransformer) // Creates NewsItemsPresentationModel
              .subscribeOn(Schedulers.io())
              .observeOn(AndroidSchedulers.mainThread())
              .subscribe(new Subscriber() {
                  public void onNext(NewsItemsPresentationModel pm){
                      view.setNewsItems(pm);
                      view.showContent();
                  }
                  public void onError(Throwable t){
                    view.showError(t);
                  }
              });
  }
}
{% endhighlight %}

As you can see, with RxJava we can use `map()` operator to transform the model into a presentation model. Another nice thing to note is that this transformation and the calculation of the difference runs on the background thread (Schedulers.io()).

The View can now be very stupid:

{% highlight java %}
class NewsItemsActivity extends Activity implements NewsItemView, OnRefreshListener {


  @Bind(R.id.recyclerView) RecyclerView recyclerView;  
   NewsItemsPresenter presenter;
   AdapterCommandProcessor commandProcessor;

   @Override
   protected void onCreate(Bundle b){
     super.onCreate(b);
     setContentView(R.layout.activity_newsitems);

     presenter = new NewsItemsPresenter();
     Adapter adapter = new NewsItemsAdapter();
     commandProcessor = new AdapterCommandProcessor(adapter);

     recyclerView.setAdapter(adapter);
     presenter.loadItems();
   }

   @Override
   public void onRefresh(){
     presenter.loadItems();
   }

   @Override
   public void setNewsItems(NewsItemsPresentationModel pm) {
     adapter.setItems(pm.newsItems);
     commandProcessor.execute(pm.commands);
   }

   ...
}
{% endhighlight %}

Hopefully, you see now that the View is pretty dumb, easier to maintain and to test.

# Behind the scenes
You might think that you don't need a third party library to do that. Actually, this is true for simple use cases where you know that lists are chronological ordered and items from the new list will be always added on top (or at the end) of the old list. In that case you simply would write `diff = newList - oldList` and then call `adapter.notifyItemRangeInserted(0, diff.size())`, right? Also in this use case you could use this library simply to not write command classes and command processor again by yourself. But what if you implement such a newspaper app as described above and news items title can be changed so that `adapter.notifyItemChanged()` has to be called? Or what if lists are not always sorted the same way?

In that case `DiffCommandsCalculator` is the drop in solution. But how does it actually works?
Let's compare two lists:

{% highlight java %}
  oldList     newList
    A           A
    B           B
    C           B2
    D           C
    E           F
    F           G
    G           E
                H
{% endhighlight %}

Basically we have inserted `B2`, removed `D` and moved `E` and inserted `H` at the end of the list:

{% highlight java %}
> B2  2
 < D   3
 < E   4  
 > E   6  
 > H   7
{% endhighlight %}

The first columns indicates whether it was an insertion `>` or a deletion `<`. The second column is the item and the last column the index of the list item (beginning by zero). This schemes seems familiar, doesn't it? You see something like almost everyday if you use `git` and `diff` (command line tool, GUIs also available) to detect and resolve merge conflicts. `DiffCommandsCalculator` basically implements the same algorithm as `diff`. This kind of problem is called [longest common subsequence problem](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem). On arbitrary number of input data solving this problem is [NP-hard](https://en.wikipedia.org/wiki/NP-hardness). Fortunately, we have fixed size of list and items count. Therefore, we can implement an algorithm that uses the concept of [dynamic programming](https://en.wikipedia.org/wiki/Dynamic_programming) that solves this problem in polynomial time `O(n*m)` (where n is the number of elements in oldList and m the number of elements in newList). That sounds really theoretically, right? Actually, it is simpler to implement than you might have thought. I found [this youtube video](https://www.youtube.com/watch?v=P-mMvhfJhu8) helpful.

# Summary
This little library called `AdapterCommands` is available on maven central and the source code can be found on [Github](https://github.com/sockeqwe/AdapterCommands). This library is the little brother of [AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates) (favor composition over inheritance) and helps you to animate dataset changes in your RecyclerView by implementing the `commamnd pattern`. Moreover, this works quite nice with `MVP` and `PresentationModel` as shown in this blog post. **Keep in mind that the runtime of  is `O(n*m)` and therefore you should consider run `DiffCommandsCalculator` on a background thread** if you have many items in your dataset. RxJava offers a nice threading model and plays very nice into MVP and presentation model as shown above and is therefore my recommendation of how to connect all the things together.


_N.B. In Artem Zinnatullin's podcast "The Context" I said that I don't do a lot of functional UI testing, because I don't see the need to do so. My argument was that I implement my apps according MVP and my Views are pretty dumb, so there can't go much wrong. Using `AdapterCommands` emphasizes this thesis because **I do test my Presenters and PresentationModel**. Furthermore, since the `AdatperCommands` library itself is already tested I can relay on that and have one less test to write in my app. **However, that doesn't mean that you should not write functional UI Tests!** I would write functional UI Tests (i.e. with Espresso) if compiling and executing this tests wouldn't take minutes. It wouldn't hurt to test the view layer too, even if the View is dumb and there can't go much wrong. I believe in TDD. However, when a test takes more than 10 seconds to execute, the whole TDD workflow and productivity gets destroyed. Hence I ensure that from Presenter downwards everything is tested and that the dumb View gets an optimized presentation model so that there can't go much wrong_
