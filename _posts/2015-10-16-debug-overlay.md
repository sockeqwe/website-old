---
layout: post
published: true
title: Debug Overlay
mathjax: false
featured: false
comments: true
headline: Debug Overlay
categories:
  - Android
tags: [android, ui, ux, RecyclerView]
---

Lately I was looking for a way to display some app internal information to an external staff  that is not a developer. I haven't found a library that fit my needs. Hence I have written my own tiny library called `debugoverlay` to do this job.

Basically I was looking for a way to display some app internal debug information on screen for other non developers you typically would log to logcat `Log.d()`. In my concrete use case I was looking for a way to display the tracking information on screen to allow external staff (not developers) to validate that the app was tracking the right information as the string to be tracked is computed dynamically (plenty of parameters and conditions) for each screen.

Obviously, we can and should write unit tests for that (Robolectric powered unit tests could do that job much faster as instrumentation tests). However, there is a lot of money depending on the correctness of the tracking (advertisement contracts are negated on the reach of the app) and therefore an external staff has to verify manually that the tracking works correctly. As already said this external staff is not a developer. Hence he need a comfortable way to get displayed this information directly in the app. So how could we display the tracking information: The simplest way is to display a `Toast`, but that doesn't work when navigating fast in the app like swiping in a `ViewPager`. An alternative could be to add a `TextView` dynamically on each Activity and Fragments's root layout. The problem with that approach this leads to fragile code.

What I really want is having a independent window just for displaying that tracking information in a logcat alike fashion, something like this:

![Debug Overlay](/images/debugoverlay.png)

 How to implement such a window? It's easier than you might have thought. I guess you have already heard or already used [facebook's chat heads](https://www.facebook.com/help/android-app/101495056700254?rdrhc). I'm sure there are plenty tutorials and blog post out there describe how to implement something similar. In a nutshell: Implement your own android `Service` and access the `WindowManager` to add a View. Yes, services can have Views:

{% highlight java %}
WindowManager windowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams(
     LayoutParams.MATCH_PARENT, LayoutParams. WRAP_CONTENT,
            WindowManager.LayoutParams.TYPE_PHONE,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT);

windowManager.addView(listView, windowParams);
{% endhighlight %}

I used the same idea to put a `ListView` in an dedicated window and a service to add log messages to that `ListView`. I have also added a close button to remove the `ListView`:
{% highlight java %}
windowManager.removeView(listView)
service.stopSelf();
{% endhighlight %}

At the end I put this simple service in a library and create a facade `DebugOverlay` as public API responsible to start the service and send a new log message to this service. So from any Activity or Fragment we can use the `DebugOverlay` like this:

{% highlight java %}
DebugOverlay.with(context).log("Hello World");
{% endhighlight %}

Now you may ask how do I ensure that this is just added to debug builds and not to our release builds. I have packed a second library artifact with the same `DebugOverlay` facade API but does nothing on method calls. Now I can use it in our apps with gralde build types to achieve that: {% highlight groovy %}
debugCompile('com.hannesdorfmann:debugoverlay:0.2.0') // Starts the service and displays the overlay
releaseCompile('com.hannesdorfmann:debugoverlay-noop:0.2.0') // Does nothing
{% endhighlight %}

The `debugoverlay-noop` adds 1 class and 3 methods to your realase dex file, which from my point of view is an acceptable compromise while the real `debugoverlay` is used only in the debug builds.

You can find this tiny library on [github](https://github.com/sockeqwe/debugoverlay) and is available on maven central.
