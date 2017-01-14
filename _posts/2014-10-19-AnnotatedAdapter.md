---
layout: post
published: true
title: AnnotatedAdapter
mathjax: false
featured: false
comments: true
headline: AnnotatedAdapter
categories:
  - android
tags: [android, annotation-processing]
---

With [FragmentArgs](http://hannesdorfmann.com/android/fragmentargs) and [ParcelablePlease](http://hannesdorfmann.com/android/ParcelablePlease) I have already shown that **Annotation Processor** is really helpful to speedup development by reducing writing boilerplate code. Regarding Android I found one scenario where I find myself writing nearly the same code ever and ever again. I'm looking at you **Adapter** with your ViewHolders, layout inflating code and view types. In this blog post I want to announce [AnnotatedAdapter](https://github.com/sockeqwe/AnnotatedAdapter)

In nearly every app you will find a **RecyclerView** (or **ListView** or **GridView**). Under the hood Adapter classes are used to inflate and fill the view cell of **RecyclerView** and to bind the data to it's graphical representation (view cell) within the RecyclerView. There are serval things you as developer have to implement:

 1. **Inflate xml layout**
 2. Create a **ViewHolder** class and write **findViewById()** for every subview in it
 3. **Bind the model** to the view holder

Furthermore not every view cell should look the same and you probably have multiple layouts for your view cells. So called **view types**, represented as integer constant in adapters code, are used to distinguish different type of view cells. Fortunately in **RecyclerView** (unlikely **ListView**) you don't have to implement the recycling part by hand. However, you have to implement those three steps from above (infalte layout, ViewHolder, bind model) for each view type in your adapter.

#AnnotatedAdapter
At this point I started thinking to myself: _"could that be generated with an annotation processor?"_ Finally I came to the conclusion: _Yes it can_ and here is how this will be done by using **AnnotatedAdapter**.

Lets assume we want to display two different view cells in a RecyclerView, one with just a TextView and one with a TextView and an ImageView (icon). I know for such simple layouts we would not need two different view types but could simply change ImageViews visibility. But lets keep this example simple. I just want to give you an idea of how AnnotatedAdapter works. So our model class looks like this;

{% highlight java %}
class Item {
  public String text;
  public int iconRes = 0; // R.drawable.foo

  public boolean hasIcon(){
    return iconRes > 0;
  }

}
{% endhighlight %}

If **Item** has no icon then the iconRes will be **0**. In this case we want to use the view type and layout with just one TextView. Otherwise we use the other layout (TextView and ImageView). The xml layouts look like this:

{% highlight xml %}
<FrameLayout
  android:layout_width="match_parent"
  android:layout_height="wrap_content">

    <TextView
      android:id="+@id/textView"
      android:layout_width="match_parent"
      android:layout_height="wrap_content" />

</FrameLayout>
{% endhighlight %}

{% highlight xml %}
<LinearLayout
  android:layout_width="match_parent"
  android:layout_height="wrap_content"
  android:orientation="horizontal">

    <ImageView
      android:id="+@id/iconImageView"
      android:layout_width="20dp"
      android:layout_height="20dp" />

    <TextView
      android:id="+@id/textView"
      android:layout_width="0dp"
      android:layout_height="wrap_content"
      android:layout_weight="1" />

</LinearLayout>
{% endhighlight %}

Every Adapter in **AnnotatedAdapter** must extend from **SupportAnnotatedAdapter** (for RecyclerView) or **AbsListAnnotatedAdapter** (for AbsListView widgets like ListView or GridView). This important because in this base adapters the generated code will be sticked together with your handwritten data-binding code and the internal view cell recycling mechanism.

Let's have a look at the complete Adapter by using AnnotatedAdapter. Afterwars we will go through the code line by line:

{% highlight java %}
public class SampleAdapter extends SupportAnnotatedAdapter
                          implements SampleAdapterBinder {

  @ViewType(
    layout = R.layout.cell_simple,
    views = @ViewField(
        type = TextView.class,
        name = "text",
        id = R.id.textView
        )
  )
  public final int simpleRow = 0;


    @ViewType(
      layout = R.layout.cell_with_icon,
      views = {
        @ViewField(
          type = TextView.class,
          name = "text",
          id = R.id.textView
          ),
        @ViewField(
          type = ImageView.class,
          name = "icon",
          id = R.id.iconImageView
          )
        }
    )
    public final int iconRow = 1;



  private List<Item> items;

  public SampleAdapter(Context context, List<Item> items) {
    super(context);
    this.items = items;
  }

  @Override
  public int getItemCount() {
    return items == null ? 0 : items.size();
  }

  @Override
  public int getItemViewType(int position) {
    if (items.get(postion).hasIcon())
        return iconRow
    else
        return simpleRow;
    }
  }

   @Override
   public void bindViewHolder(SampleAdapterHolders.IconRowViewHolder vh,
        int position) {

      Item item = items.get(position);
      vh.text.setText(item.text);
      vh.icon.setImageResource(item.icon)
    }

    @Override
    public void bindViewHolder(SampleAdapterHolders.SimpleRowViewHolder vh,
         int position) {

       Item item = items.get(position);
       vh.text.setText(item.text);
     }
}
{% endhighlight %}

At first look you have noticed that there is less code. However AnnotatedAdapter can't read your mind. Therefore we have to give some information by using **@ViewType** annotation. In short: less code, more annotations. With the **@ViewType** annotation we specify the view types integer constant, the xml layout we want to inflate for this view type and with **@ViewField** you define the view holder class. Have a look at the **int iconRow = 1**:
{% highlight java %}
@ViewType(
  layout = R.layout.cell_with_icon,
  views = {
    @ViewField(
      type = TextView.class,
      name = "text",
      id = R.id.textView
      ),
    @ViewField(
      type = ImageView.class,
      name = "icon",
      id = R.id.iconImageView
      )
    }
)
public final int iconRow = 1;
{% endhighlight %}

AnnotatedAdapter will generate a view holder class called **IconRowViewHolder** that looks like this:

{% highlight java %}
class IconRowViewHolder extends RecyclerView.ViewHolder {

  public TextView text;
  public ImageView icon;

  public RowWithPicViewHolder(View view) {

     super(view);

     text = (TextView) view.findViewById(R.id.textView);
     icon = (ImageView) view.findViewById(R.id.iconImageView);
   }

}
{% endhighlight %}

 - **@ViewField ( type = ... )** specifies the class of the view in the view holde
 - **@ViewField ( name = ... )** specifies the name of the field for this subview in the viewholder
 - **@ViewField ( id = ... )** specifies the id of to get the subview instance by using **findViewById()**


You can also specify non UI related fields (not bound by **findViewById()** ), for instance a **OnClickListener**, as part of the generated view holder. You can do that by using **@Field** like this:

{% highlight java %}
@ViewType(
  layout = R.layout.cell_with_icon,
  views = {
    @ViewField(
      type = TextView.class,
      name = "text",
      id = R.id.textView
      ),
    @ViewField(
      type = ImageView.class,
      name = "icon",
      id = R.id.iconImageView
      )
    },
  fields = {
    @Field(
      type = OnClickListener.class,
      name = "clickListener"
      )
    }

)
public final int iconRow = 1;
{% endhighlight %}

Then the generated **IconRowViewHolder** contains this **@Field** as well:

{% highlight java %}
class IconRowViewHolder extends RecyclerView.ViewHolder {

  public TextView text;
  public ImageView icon;

  public OnClickListener clickListener;

  public RowWithPicViewHolder(View view) {

     super(view);

     text = (TextView) view.findViewById(R.id.textView);
     icon = (ImageView) view.findViewById(R.id.iconImageView);
   }

}
{% endhighlight %}



Next you see two methods that you already know if you ever have written an Adapter by hand:

 - **getItemCount()**: Returns the total number of items in the data set hold by the adapter
 - **getItemViewType(int position)**: Return the view type of the item at position for the purposes of view recycling

Last you notice that there are **bindViewHolder()** methods. In this methods you bind the data item to the view cell by using the corresponding ViewHolder. Where does this methods come from? This methods are defined in the interface **SampleAdapterBinder**. This interface is generated by AnnotatedAdapter by evaluating the **@ViewType** annotation. Hence **SampleAdapter** has to implement the interface **SampleAdapterBinder**.

 > That's all? That sounds to good to be true?

Yes, that's all, **but** there is one thing you have to know about android studio (I assume the most of you are using Android Studio): Since the interface **AdapterBinder** is generated at compile time this interface is not available when you start creating a brand new Adapter. The interface is  available only after compiling at least once. Hence I would consider the following step as best practice when creating a brand new adpater class using **AnnotatedAdapter**:

1. Create the new java file for your adapter and let the adapter class extends **SupportAnnotatedAdapter**.
2. Define your view types by using **@ViewType** and **@ViewField**
3. In Android Studio: Main Menu Bar -> **Build -> Rebuild Project**. This will force the compiler to  generate the **AdapterBinder** interface.
4. Let your adapter class implement the binder interface and implement all binder methods

I can guaranty that by following this instructions you will write your next adapter in less than 5 minutes (creating xml layouts excluded). **Note** that you have to trigger **Build -> Rebuild Project** only at the very first time you create a brand new adapter class. Once done you can change your adapters code, you can add or remove view types and the interface will be regenerated automatically every time you compile/install your app on your device.

# I need your help
From my point of view **AnnotatedAdapter** reduces writing boilerplate code a lot. However, it's not perfect, since you have to use **@ViewField** annotations to define the ViewHolder. I want to remove this  and make AnnotatedAdapter by parse the xml layout files to retrieve all needed informations to generate the view holder classes. Unfortunately that's not as simple as it sounds, but I'm sure that there are smarter people than me out there. If you have an idea of how to solve that problem please use [this issue on github](https://github.com/sockeqwe/AnnotatedAdapter/issues/4) to get in touch with me.


AnnotatedAdapter supports also **AbsListView** widgets, inheritance and other little features you may find useful. You can find more information about [AnnotatedAdapter on github](https://github.com/sockeqwe/AnnotatedAdapter)
