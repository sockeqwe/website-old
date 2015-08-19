---
layout: post
published: true
title: Don't interrupt the user's flow
mathjax: false
featured: false
comments: true
headline: Don't interrupt the user's flow
categories:
  - Android
tags: [android, ui, ux, RecyclerView]
---

From my point of view user experience is a very important topic but sometimes doesn't get the attention it deserves. In this blog post I want to show you how to use **RecyclerView** to build an outstanding user experience by not interrupting the user's flow.

I'm very lucky to work in a very talented team at [Tickaroo](https://www.tickaroo.com) where we are building and maintaining the android and iOS apps for kicker, one of the most important football magazine in Europe (even though the main language is German). The app basically displays news about football (but also about other sports), live scores & live updates (push notifications), statistics, videos, photo galleries and so on and so forth. A few days ago we published an update for the [kicker android app](https://play.google.com/store/apps/details?id=com.netbiscuits.kicker) where we integrated a new interactive feature: tip game (betting game, footy tipping, tip guessing or whatever the word for that is in your local area). So the idea is that the users of the kicker app try to guess the results of football games and get points when the guessed results are correct.

When starting the kicker app you see a `RecyclerView` displaying different items, amongst others news articles, game results and upcoming games. The first question is how do we integrate the new tip game feature? Next, how does the user can submit "tips" (try to guess the result) for a game? We could have made our life easy (from developers point of view) and just start a new Activity to display a list of games the user can submit "tips" (try to guess the result) for. But we decided to implement two _"modes"_ as we already display the games the user can guess results in the RecyclerView: The _"normal"_ mode where our users see the upcoming games or results of the last games and the _"tip"_ mode where users can submit a tip (guessthe result) and see if their guess was correct after the game is finished. So the final result looks like this:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/7XAe3zd0fdU" frameborder="0" allowfullscreen></iframe>
</p>

Instead of interrupting the users flow by starting a new Activity we decided to switch between both modes with a flip animation by pressing the "tippen" (german for "guessing results") and "schließen" (german for "close") button.

How did we implement that? Obviously I can't share the whole source code, but I will give you some insights and pitfalls we have faced.

## Flip items
As you have seen in the video shown above, we are animating the items of the `RecyclerView`. We started by using a layout file containing both child layouts, normal mode and tip mode, like this:
 {% highlight xml %}
 <FrameLayout>
    <LinearLayout
      android:id="normalMode"
      android:layout_width="match_parent"
      android:layout_height="match_parent">
        ...
    </LinearLayout>

    <LinearLayout
      android:id="tipMode"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:visibility="invisible">
        ...
    </LinearLayout>

 </FrameLayout>
 {% endhighlight %}

 Then the ViewHolder in the RecyclerView will have a reference to _"normalMode"_ and _"tipMode"_ layout. Flipping the views is simply a `rotationX()` animation like as shown below (don't forget to set the visibility from "visible" to "invisbible" and vice versa):
{% highlight java %}
public void animateToTipMode(TipViewHolder holder, int delay){
 int duration = 100;
 // Flip the "normalMode" View "out"
 ViewCompat.animate(viewHolder.normalMode)
          .rotationX(90) // Animate from 0 to 90
          .setDuration(duration)
          .setStartDelay(delay)
          .setListener(new ViewPropertyAnimatorListenerAdapter(){
              @Override public void onAnimationEnd(View view) {
                viewHolder.normalMode.setVisibility(View.INVISIBLE);
              }
            })
          .start();

  // Flip the "topMode" View "in"
  ViewCompat.setRotationX(holder.tipMode, -90);
  ViewCompat.animate(viewHolder.tipMode)
           .rotationX(0) // Animate from -90 to 0
           .setDuration(duration)
           .setStartDelay(delay)
           .setListener(new ViewPropertyAnimatorListenerAdapter(){
               @Override public void onAnimationEnd(View view) {
                 viewHolder.normalMode.setVisibility(View.VISIBLE);
               }
             })
           .start();
}
{% endhighlight %}

So basically the animation runs from 0 to 90 for animating view out and from -90 to 0 for animating the view in. The wave alike execution of the flip animation is done by adding a start delay and increase the start delay for each cell. So all we have to do is to collect those items above the button that triggers the switch betewen both modes. `TipViewHolder` is the ViewHolder for the xml layout containing normalMode and tipMode child layouts. In the old days of `ListView` we would have done that simply by iterating over its children with `ListView.getChildAt(index)`, but `RecyclerView` internally handles its children different (with a LayoutManager). Hence, `RecyclerView.getChildAt(index)` doesn't return the views as the `ListView`does.
When working with `RecyclerView` you have to work with `ViewHolder` as well. `ViewHolder` is not just a plain old class to hold references to the child views and reduces `findViewById()` operations, no, `ViewHolder` knows more about how its used internally used in the parent `RecyclerView`. So you can query the current adapter position of a ViewHolder with `ViewHolder.getAdapterPosition()`. We use that to get the adapter position of the button that starts the flip animation (have a look at the video shown above: there is a button to start the flip animation to switch between "normal mode" and "tip mode"). In our app we know that all  games (`TipViewHolder`) are above the clicked button, so we just have to iterate from button position up in adapters dataset:
 {% highlight java %}
 public void onSwitchToTipModeClicked(ViewHolder buttonViewHolder){

   int adapterPosition = buttonViewHolder.getAdapterPosition();
   int index = adapterPosition - 1;
   int delay = 70;

   while (index >= 0){
     ViewHolder vh = recyclerView.findViewHolderForAdapterPosition(index);

     if (! (vh instanceof TipViewHolder) ){
        break;
     }

       animateToTipMode((TipViewHolder) vh, delay * (adapterPosition - index));
       index --;
   }

 }
{% endhighlight %}

We use to `viewHolder.getAdapterPosition()` to determine the start position and then we step up in the adapter and check if the ViewHolder at the given index is of type `TipViewHolder`. We can get the ViewHolder at the given adapter position by using `RecyclerView.findViewHolderForAdapterPosition(index)`. Note that this returns `null` if there is no ViewHolder for that adapter index because the item with the given adapter index is not visible on screen (item is above or below the visible rectangle of RecyclerView, the user have to scroll to bring that item into RecyclerViews visible rectangle). Please note also that `null instanceof Something` returns false.

Another thing developers should take more care of when building their app is that animations can be used to "hide" that your app is loading data. In our app when the user switches from "normal mode" to "tip mode"  we execute a http request to load the previously guessed results (if there are any) in background. Usually, our users don't notice that data has been loaded because we don't show a loading indicator. We use the time the wave alike animation takes to load the data. This gives the user of our app the impression that our app works lightning fast because there is no visual interruption by displaying a loading indicator. What if the user has a slow internet connection? Then yes, we display a loading indicator after the wave animation is finished.

## The downside
What is the downside of this approach? The layout file contains a huge view hierarchy since it displays both "normalMode" and "tipMode". That wouldn't be a problem if the majority of our users would use high level devices like a Galaxy S6 or even devices comparable with the oldie but goldie Nexus 4. Unfortunately it turns out that decent devices like the Samsung Galaxy S3 Mini are the most used devices of our users. What does that mean? Huge view hierarchy leads to bad scroll performance. The tip game is an optional feature of our app mainly used by power users. I, personally, expect that about 30% of our users will participate in the tip game. That would mean that 70% of the users may face scroll performance issues because of the tip game even if they don't participate in the tip game at all. Hence we decided to implement the flip animation in a different way: Instead of having one single xml layout for both modes (normal and tip mode) we decided to split them into two layout files and define two view types in our adapter.

**Disclaimer:** This might be an overkill if your layout files are already flat enough then the solution described above should just work. However, for us it was necessary to work with different view types to guarantee good scroll performance even on decent devices.

Let me explain you how we have implement the same wave alike animation as described in the previous solution: Fortunately, RecyclerView is really powerful and open for extensions. We have implemented a custom `ItemAnimator` that executes the flip animation (still the same rotationX animation as shown before, from 0 to 90 and -90 to 0). Writing a custom `ItemAnimator` is not rocket science but it takes some time to dive into the methods you have to implement to make the `ItemAnimator` work properly. You may be confused why we need an `ItemAnimator`. An ItemAnimator allows you do define animations to run when new items are inserted, moved or removed from RecyclerViews Adapter. So what we do is we modify the adapters dataset. The idea is the same as shown in the previous solution: we iterate over the adapters dataset, but this time instead of checking for the `ViewHolder` and starting the flip animation manually we remove the `Game` items (displaying items in "normal mode") from adapters dataset and replace them with `Tip` items (displaying items in "tip mode"). How is it related to `ItemAnimator`? If you haven't noticed yet, RecyclerView offers two methods `notifyItemRangeRemoved()` and `notifyItemRangeInserted()`:

{% highlight java %}
public void onSwitchToTipModeClicked(ViewHolder buttonViewHolder, List dataset){

  int MIN_DEVICE_YEAR = 2012;

  int adapterPosition = buttonViewHolder.getAdapterPosition();
  int index = adapterPosition - 1;
  int bottomIndex = index;
  int invisibleStartIndex = - 1;

  List<Tip> tipsToInsert = new ArrayList<>();

  while (index >= 0){
    Object item = dataset.get(index);

    if (! (item instanceof Game) ){
       break;
    }

    Game game = (Game) item;

    RecyclerView.ViewHolder viewHolder =
            recyclerView.findViewHolderForAdapterPosition(index);

    if (viewHolder == null){
      // Element not visible on screen, so we can replace the item directly
      dataset.set(index, game.getTip());

      if (invisibleStartIndex == -1){
        invisibleStartIndex = index;
      }

    } else {
      // Element is visible on screen
      viewHolder.itemView.setTag(R.id.adapterIndexWorkaround, adapterPosition); // Workaround
      tipsToInsert.add(0, game.getTip()); // Keep the order
    }

    index--;
  }

  int topIndex = invisibleStartIndex >=  0 ? invisibleStartIndex :  index + 1 ;

  // Remove old visible "Game" items
  for (int i = bottomIndex; i >= topIndex; i--) {
      dataset.removeItem(i); // Reason for workaround
  }

  if (deviceYear >= MIN_DEVICE_YEAR){
    adapter.notifyItemRangeRemoved(topIndex, tipsToInsert.size()); // Triggers remove flip animation
  }

  // Insert visible "Tip" items
  datase.addAll(topIndex, tipsToInsert);

  if (deviceYear >= MIN_DEVICE_YEAR){
    adapter.notifyItemRangeInserted(topIndex, tipsToInsert.size()); // Triggers insert flip animation
  } else {
    // Without animations because of low-end device
    adapter.notifyDataSetChanged();
  }

}
{% endhighlight %}

Let's discuss the code shown above: Similar to the first discussed solution we start at the adapter position of the button to switch between the normal and tip mode. Since we can assume that the button is visible in the RecyclerView (otherwise it couldn't be clicked right?) and all `Games` are above the button we just have to iterate from bottom to top in adapters dataset. We use the check `item instanceof Game` to determine where the group of games finishes (in our app we always have at least one different item between a group of games). Next we check if `viewHolder == null` which means that the item is not visible in RecyclerView. In that case we can directly replace the `Game` item with the corresponding `Tip` item. We also note the index where the first not visible item begins by storing the adapter index into `invisibleStartIndex`. If the item is visible then we add the corresponding tip into `tipsToInsert`. We also have to implement a little workaround to keep the original adapter's position by using `setTag()` which allows to store data in an internally `Map<Integer, Object>` every `android.view.View` has (it's recommended to use `R.id.something` as key). Why do we need this workaround? Well, we want that our `ItemAnimator` flips the items in a wave alike manner. To schedule the single animations for each item correctly it's important to know the index. Every `ViewHolder` knows it's index by using `viewHolder.getAdapterPosition()`, but that method will return `-1` as we have removed the item from adapters dataset, which is the desired  by design. Usually `ItemAnimator` can use `viewHolder.getOldPosition()` which would return the previous position (before removing item from adapters dataset). But the problem is that in our concrete use case all `viewHolder.getOldPosition()` return the same value. Why? Because we are removing every item manually in a for loop (because `List` lacks a method to remove a multiple items at once similar to `list.addAll()`). Hence, our `ItemAnimator` have to use that workaround to schedule the flip animation properly. Last, you may have noticed `MIN_DEVICE_YEAR`. I can't remember what the problem exactly was (I should note down such things), but on very old devices you can run into the problem that the RecyclerView reaches an insane internal state (which could lead to crashes) if the item animation takes to long. To avoid that we use [Facebook's Device Year Class](https://github.com/facebook/device-year-class) library to detect old devices and simply don't run the flip animation on those devices by directly calling `adapter.notifyDataSetChanged()`.

## Submit a tip
Next you may be wondering how you can submit a tip. We started by implementing a dialog with number pickers. But the problem was that you have to open the dialog for a game to submit a tip, close the dialog and open the next tip dialog for the next tip. This was definitely not the user experience we wanted. So we improved that by passing a list of games the user can tip to a single dialog to avoid that open and close for each:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/5Vk6ZLNm-l8" frameborder="0" allowfullscreen></iframe>
</p>


## Guessing in a rush
However, a Dialog is still interrupting the user experience. There must be a better way! So we came up with the idea to use swipe gestures. This allows the user to guess results on the fly without interrupting the user's flow by displaying a Dialog. Have a look at the final result:

<p>
<iframe width="420" height="315" src="https://www.youtube.com/embed/ZOXh1KWzTWk" frameborder="0" allowfullscreen></iframe>
</p>

By swiping the item to the right the user can increase the score of the home team while swiping the item left increases the away team score. Implementing that is also straight forward if you have ever worked with `MotionEvents` on Android. In android there are basically two methods that get called during dispatching MotionEvents: In `boolean onInterceptTouchEvent()` you get some MotionEvents to decide if that view should intercept and consume all MotionEvents afterwards in `void onTouchEvent()`. It's like ordering a good bottle of wine in a restaurant. The waiter brings you the bottle and a wine glass and offers you a sip of the ordered wine . After taking a small sniff, inspecting the color, swirl the wine in your glass you decide whether to take the wine or not (returning true or false in `onInterceptTouchEvent()`). If yes, you consume the wine (`onTouchEvent()`). Working with `RecyclerView` is not different. Fortunately, you don't have to override that methods of RecyclerView because those methods are used to detect scroll and fling events. RecyclerView offers you to add an `OnItemTouchListener` to intercept `MotionEvents` for items displayed in the RecyclerView.

{% highlight java %}
public abstract class TipSwipeListener implements RecyclerView.OnItemTouchListener {

  private final int threshold;
  private float xStart = 0;
  private float yStart = 0;
  private TipViewHolder startViewHolder;

  private int increaseMaxSwipeDistance;

  @Override public boolean onInterceptTouchEvent(RecyclerView rv, MotionEvent e) {

    final int action = MotionEventCompat.getActionMasked(e);

    // Touching somewhere in the RecyclerView
    if (action == MotionEvent.ACTION_DOWN) {
      xStart = e.getX();
      yStart = e.getY();

      // Determine the View we touch
      View startTouchedView = rv.findChildViewUnder(e.getX(), e.getY());
      if (startTouchedView == null) {
        return false;
      }

      // Determine the ViewHolder that has been touched
      RecyclerView.ViewHolder vh = rv.getChildViewHolder(startTouchedView);

      if (vh != null && vh instanceof TipViewHolder) {
        startViewHolder = (TipViewHolder) vh;
      }
      return false;
    }

    // We begin to move the finger on the screen (not released finger from screen)
    if (action == MotionEvent.ACTION_MOVE && startViewHolder != null) {

      float xDif = Math.abs(xStart - e.getX());
      float yDif = Math.abs(yStart - e.getY());

      if (xDif >= threshold && xDif > yDif) {
        // finger is moving horizontally
        return true;
      }

      // finger is moving vertically
      return false;
    }

    // releasing finger from screen
    if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
      reset();
    }

    return false;
  }

  @Override public void onTouchEvent(RecyclerView rv, MotionEvent e) {

    final int action = MotionEventCompat.getActionMasked(e);

    float xDif = e.getX() - xStart;

    if (action == MotionEvent.ACTION_MOVE) {
      if (startViewHolder.getTippView() != null) {

        if (xDif < 0) {
          startViewHolder.translateXAwayTeam(0);
          startViewHolder.translateXHomeTeam(Math.max(xDif, -increaseMaxSwipeDistance));
        } else if (xDif > 0) {
          startViewHolder.translateXHomeTeam(0);
          startViewHolder.translateXAwayTeam(Math.min(xDif, increaseMaxSwipeDistance));
        } else {
          startViewHolder.getTippView().translateXHomeTeam(0);
          startViewHolder.getTippView().translateXAwayTeam(0);
        }
      }
    } else
      // Released or Canceled swipe gesture
      if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
        if (xDif < -increaseMaxSwipeDistance) {
          onIncreaseAwaySwipe(); // Swiped far enough
        } else if (xDif > increaseMaxSwipeDistance) {
          onIncreaseHomeSwipe(); // Swiped far enough
        } else {
          animateToStartState(xDif);
        }
      }
  }


  private void onIncreaseAwaySwipe() {
    ...
    reset();
  }

  private void onIncreaseHomeSwipe() {
    ...
    reset();
  }


  private void animateToStartState(float xDif) {
    // Just animate the X translation back to 0
    ...

    reset();
  }

  private void reset() {
    xStart = 0;
    yStart = 0;

    startViewHolder = null;
  }

}
{% endhighlight %}

Most of the code should be really self explaining. We simply start in `onInterceptTouchEvent()` by checking if the user is touching on a `Tip` Item by querying the touched view  `recyclerView.findChildViewUnder(x, y)` and then check if the corresponding ViewHolder is a `TipViewHolder` by using `recyclerView.getChildViewHolder(view)`. Next we check if the user is moving his finger to left or right. Please note that `onInterceptTouchEvent()` is called multiple times and we need to inspect more than `MotionEvent` passed as parameter to this method to determine if the user is moving his finger to left or right (wine tasting). If the finger has been moved on x-achsis beyond a threshold, we return `true` to claim that we want continue to consume this gesture in `onTouchEvent()`.

In `onTouchEvent()` we simply set `translationX` property according the distance and direction the user has moved his finger (max distance is increaseMaxSwipeDistance which is half of the width of the box displaying the guessed result). And where does the _+1_ comes from while swiping? This is just a TextView which always was there hidden behind the box displaying the guessed result. By translating the view on x-achsis this TextView becomes visible. Actually, in our app we have overridden `onDraw()` of a custom parent layout to keep the layout hierarchy flat.

Adding an `ItemTouchListener` to a RecyclerView is quite easy: `recyclerView.addOnItemTouchListener()`. Last, you may be wondering if this horizontal swipe listener can be used in a `ViewPager`. It turns out that this is easier than expected. ViewPager, like any other ViewGroup allows it's children to claim that they don't want that a parent intercepts touch events by calling [ViewPager.requestDisallowInterceptTouchEvent(boolean)](https://developer.android.com/reference/android/view/ViewGroup.html#requestDisallowInterceptTouchEvent(boolean)). So a good place to set `requestDisallowInterceptTouchEvent()` to true would be in `onInterceptTouchEvent()` right before returning true and release it in `reset()` called when finger has been moved up. By the way, if you don't know if you have a ViewPager in your view hierarchy you can iterate recursively from bottom to top by using `view.getParent()` (do it just once and not every time you move your finger).

A disadvantage of our swipe approach is that the user can not decrease the guessed score since both x-achsis directions are already used. So we decided to keep the Dialog for that use case and for those not nimble-fingered users.

Another advantage of using different view types for both modes is that we can compose adapters by using [AdapterDelegates](hannesdorfmann.com/android/adapter-delegates/) (favor composition over inheritance) and so we are able to bring the flip animation to switch between normal and tip mode to alomost all RecyclerViews in our whole app without having code clones.

## Conclusion
The less the user's flow is interrupted, the better their experience! Sounds so simple, but may takes some time to implement because also the little details count. So ask yourself: Do you want to deliver an out standing user experience or are you just to lazy to write more code than required?


_PS: The credit for this user experience goes to the whole Tickaroo team. We  have developed this user experience together. Our awesome designer has had some initial thoughts and ideas, the iOS developers put some things into and we android developer as well during prototyping._
