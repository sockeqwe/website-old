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
tags: android
imagefeature: cover3.jpg
---

Developing for Android is sometimes painful. You have to write lot of code to do simple things like setting up a Fragment. Fortunately java have supports a powerfull tool: **Annotation Processors**

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

It may have worked, but did you try to rotate your device from landscape to portrait? Your app will crash with `NullPointerException` as soon as you try to acces _id or title_ .

> It's ok, my app is locked in portrait. So I never will run into this problem.

You will! Android is a real multitasking operating system. Multiple apps run at the same time and the android os will destory activities (and the containing fragments) if memory is needed. Probably you will never notice that durring your daily app development. However you user will once the app is published in the play store. They will use multiple apps at the same time and it's likely that your app is going to be destroyed in the background. For Instance: a user of the app ones your app and _MyFrament_ is displayed on screen. Next the user will press the home button (your app is going in the background) and opens some other apps. Your app will be destroyed in background to free memory. Some time later the user wants to come back to your app. He presses the multitasking button and chooses your app to come back in the foreground. So what does Android do right now? Android restores the previous app state and restores _MyFragment_ . And that's the problem, because now the fragment tries to access _title_ which is null, because it has not been stored persistent in a bundle.

> I see, so I have to save them in onSaveInstanceState(Bundle)

The answer is **NO**. The official docs are a little bit unclear, but `onSaveInstanceState(Bundle)` should be used exactly the same way you do with `Activity.onSaveInstanceState(Bundle)`: you use this method to save the instance state "temporarly", for instance to handle screen orientation changes (from portrait to landscape and vice versa). That means it's not stored persistently which is required when the app is killed in the background and restored when the app comes in foreground again. It's pretty the same as how activities work. `Activity.onSaveInstanceState(Bundle)` is used for "temporarly" saving the instance state, while the long persistent parameters are passed through the intent extra data.

> So should I save these Fragment arguments in the Activities Intent?

No, Fragment has it's own mechanism for this. There are two methods: `Fragment.setArguments(Bundle)` and `Fragment.getArguments()` and you have to use this two methods to ensure that the arguments will be stored persistently, even if the app is destroyed and restored. But that's the painful part I have mentioned above. It's a lot of code you have to write. You have to create a `Bundle`, next you have to set the key / value pairs and finally to call `Fragment.setArguments()`. But you are not done yet. You have to read the values out of the Bundle with `Fragment.getArguments()`. Something like this:

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

I hope you can understand now what I mean with "painful". There's a lot of code you have to write for any single fragment in your application. Wouldn't it be nice if someone could write that setting and reading arguments code for you? Fortunately we can thanks to Annotation Processing. Annotation Processing allows you to generate java code for you at compile time. Note that we are not talking about evaluating annotations at run time by using reflections.

`FragmentArgs` it's a lightweight simple library that will do exactly this for you. It generates javacode for your fragment arguments. Have a look at this code:

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

If you want to use it in your android application check out the [FragmentArgs project on github](https://github.com/sockeqwe/fragmentargs) where you will find more inforamtion about the api and how to use it. FragmentArgs is available on maven central.