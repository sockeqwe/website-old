---
layout: post
published: true
title: FragmentArgs
mathjax: false
featured: false
comments: true
headline: FragmentArgs
categories:
  - android
tags: [android, annotation-processing]
---

Developing for Android is sometimes painful. You have to write lot of code to do simple things like setting up a Fragment. Fortunately java supports a powerful tool: **Annotation Processors**

The Problem with Fragments is that you have to set arguments (the parameters) for a fragment to make them work correctly. Many new android developers that write the first fragment do something like this:
{% highlight java %}
public class MyFragment extends Fragment {

  private int id;
  private String title;

  public static MyFragment newInstance(int id, String title) {
    MyFragment f = new MyFragment();
    f.id = id;
    f.title = title;
    return f;
  }

  @Override
    public View onCreateView(LayoutInflater inflater,
        ViewGroup container, Bundle savedInstanceState) {

            Toast.makeText(getActivity(), "Hello " + title.substring(0, 3),
                Toast.LENGTH_SHORT).show();
      }
}
{% endhighlight %}

> What's wrong with that? I have tested it on my device and it worked like a charm?

It may have worked, but did you try to rotate your device from portrait to landscape? Your app will crash with `NullPointerException` as soon as you try to acces _id or title_ .

> It's ok, my app is locked in portrait. So I will never run into this problem.

**You will!** Android is a real multitasking operating system. Multiple apps run at the same time and the android os will destroy activities (and the containing fragments) if memory is needed. Probably you will never notice that during daily app development. However, once the app is published in the play store you will get notified that your app is crashing and you may wonder why. Your app users use multiple apps at the same time and it's very likely that your app is going to be destroyed in the background. Example: a user of your app opens your app and _MyFrament_ is displayed on screen. Next the user will press the home button (your app is going in the background) and opens any other app. Your app will be destroyed in the background to free memory. Later on the user comes back to your app by pressing the multitasking button and your app comes back in the foreground. So what does Android do right now? Android restores the previous app state and restores _MyFragment_ . And that's the problem, because now the fragment tries to access _title_ which is null, because it has not been restore since it was not stored persistent in a bundle.

> I see, so I have to save them in onSaveInstanceState(Bundle)?

**NO**. The official docs are a little bit unclear, but `onSaveInstanceState(Bundle)` should be used exactly the same way you do with `Activity.onSaveInstanceState(Bundle)`: you use this method to save the instance state "temporarly", for instance to handle screen orientation changes (from portrait to landscape and vice versa). That means it's not stored persistently which is required when the app is killed in the background and restored when the app comes in foreground again. It's pretty the same as how activities work: `Activity.onSaveInstanceState(Bundle)` is used for "temporarly" saving the instance state, while the long persistent parameters are passed through the intent extra data.

> So should I save these Fragment arguments in the Activities Intent?

No, Fragment has it's own mechanism for this. There are two methods: `Fragment.setArguments(Bundle)` and `Fragment.getArguments()` and you have to use this two methods to ensure that the arguments will be stored persistently, even if the app is destroyed and restored. But that's the painful part I have mentioned above. It's a lot of code you have to write. First, you have to create a `Bundle`, next you have to set the key / value pairs and finally to call `Fragment.setArguments()`. But you are not done yet. You have to read the values out of the Bundle with `Fragment.getArguments()`. Something like this:

{% highlight java %}
public class MyFragment extends Fragment {

  private static String KEY_ID ="key.id";
  private static String KEY_TITLE = "key.id";

  private int id;
  private String title;

  public static MyFragment newInstance(int id, String title) {
    MyFragment f = new MyFragment();
    Bundle b = new Bundle();
    b.putInt(KEY_ID, id);
    b.putString(KEY_TITLE, title);
    f.setArguments(b);
    return f;
  }

  @Override
  public void onCreate(Bundle savedInstanceState) {
      // onCreate it's a good point to read the arguments
      Bundle b = getArguments();
      this.id = b.getInt(KEY_ID);
      this.title = b.getString(KEY_TITLE);
  }

  @Override
  public View onCreate(LayoutInflater inflater,
        ViewGroup container, Bundle savedInstanceState) {

            // No NullPointer here, because onCreate() is called before this
            Toast.makeText(getActivity(), "Hello " + title.substring(0, 3),
                Toast.LENGTH_SHORT).show();
      }
}
{% endhighlight %}

I hope you can understand now what I mean with "painful". There's a lot of code you have to write for any single fragment in your application. Wouldn't it be nice if someone else could write that code for you? Annotation Processing allows you to generate java code for you at compile time. Note that we are not talking about evaluating annotations at run time by using reflections.

##FragmentArgs

FragmentArgs is a lightweight simple library that generates that java code for your fragments. Have a look at this code:

{% highlight java %}
import com.hannesdorfmann.fragmentargs.FragmentArgs;
import com.hannesdorfmann.fragmentargs.annotation.Arg;

public class MyFragment extends Fragment {

	@Arg
	int id;

	@Arg
	String title;

	@Override
	public void onCreate(Bundle savedInstanceState){
		super.onCreate(savedInstanceState);
		FragmentArgs.inject(this); // read @Arg fields
	}

	@Override
	public View onCreateView(LayoutInflater inflater,
		ViewGroup container, Bundle savedInstanceState) {

      		Toast.makeText(getActivity(), "Hello " + title,
      			Toast.LENGTH_SHORT).show();
      }
}
{% endhighlight %}


`FragmentArgs` generates the boilerplate code for you just by annotating fields of your Fragment class. In your Activity you will use the generated `Builder` class _(the name of your fragment with "Builder" suffix)_ instead of `new MyFragment()` or a static `MyFragment.newInstance(int id, String title)` method.

For example:

{% highlight java %}
public class MyActivity extends Activity {

	public void onCreate(Bundle savedInstanceState){
		super.onCreate(savedInstanceState);

		int id = 123;
		String title = "test";

		// Using the generated Builder
		Fragment fragment =
			new MyFragmentBuilder(id, title)
			.build();

		// Fragment Transaction
		getFragmentManager()
			.beginTransaction()
			.replace(R.id.container, fragment)
			.commit();
	}

}
{% endhighlight %}

You may have noticed this line of code `FragmentArgs.inject(this);` in  `Fragment.onCreate(Bundle)`. This is the important thing where your fragment gets connected to the generated code. You may ask yourself: _"Do I have to override onCreate(Bundle) in every Fragment to add the inject() method call?"_ . The answer is no. A powerful feature is that `FragmentArgs.inject(this);` supports inheritance. Simply add this line in your _base fragment_ and make all your fragments extending this base fragment:

{% highlight java %}
public class BaseFragment extends Fragment {

    @Override
    public void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        FragmentArgs.inject(this); // read @Arg fields
    }
}

public class MyFragment extends BaseFragment {

  @Arg
  String title;

  @Override
  public View onCreateView(LayoutInflater inflater,
    ViewGroup container, Bundle savedInstanceState) {

      Toast.makeText(getActivity(), "Hello " + title,
        Toast.LENGTH_SHORT).show();
  }

}
{% endhighlight %}

If you want to learn more about `FragmentArgs` check out the [FragmentArgs project on github](https://github.com/sockeqwe/fragmentargs) where you will find more information about the API and how to use it. FragmentArgs is available on maven central.

**Credits:** Parts of the annotation processing code are based on Hugo Visser's [Bundle](https://bitbucket.org/hvisser/bundles) project.
