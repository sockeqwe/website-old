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

Lately I was looking for a way to display some app internal information to an external staff  that is not a developer. I haven't found a library that fit my needs. Hence I have written my own tiny library called `DebugOverlay` to do this job.

Usually we as developers would log app internal things by using logcat `Log.d()`. But what if we have to provide app internal logs to non developers? Can they access logcat? Do they have to install android studio and enable developer options on their android device? Wouldn't it be nice if non developers could just see these logging information directly in the app somehow?
In my concrete use case I was looking for a way to display tracking information on screen to allow external staff (not developers) to validate that the app was tracking the right information as the string to be tracked is computed dynamically for each screen (plenty of parameters and conditions).

Obviously, we can and should write unit tests for that (Robolectric powered unit tests could do that job much faster as instrumentation tests). However, there is a lot of money depending on the correctness of the tracking (advertisement contracts are negated on the reach of the app) and therefore an external staff has to verify manually that the tracking works correctly. As already said this external staff is not a developer. Hence he need a comfortable way to get displayed this information directly in the app. So how could we display the tracking information? The simplest way is to display a `Toast`, but that doesn't work well when navigating fast in the app like swiping in a `ViewPager`. An alternative could be to add a `TextView` dynamically on each Activity and Fragment's root layout. The problem with that approach is that this leads to fragile code.

What I really want is having an independent window just for displaying tracking information in a logcat alike fashion, something like this:

![Debug Overlay](/images/debugoverlay.png)

 Why an independent window? Because I want avoid side effects on my production code. How to implement such a window? It's easier than you might have thought before. I guess you have already heard or already used [facebook's chat heads](https://www.facebook.com/help/android-app/101495056700254?rdrhc). I'm sure there are plenty tutorials and blog post out there describing how to implement something like that. In a nutshell: Implement your own android `Service` and use the `WindowManager` to add a View. Yes, services can have Views:

{% highlight java %}
class DebugOverlayService extends Service {

  @Override
  public void onCreate(){
    FrameLayout rootLayout = new FrameLayout();
    rootLayout.addView(listView);
    rootLayout.addView(closeButton);

    WindowManager windowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

    WindowManager.LayoutParams windowParams = new WindowManager.LayoutParams(
            LayoutParams.MATCH_PARENT, LayoutParams. WRAP_CONTENT,
            WindowManager.LayoutParams.TYPE_PHONE,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT);

    windowManager.addView(rootLayout, windowParams);
  }
}
{% endhighlight %}

As you see, I use the chat heads idea to put a `FrameLayout` containing a `ListView` and a close button in an dedicated window. By clicking on the close button the service will be stopped and the dedicated window removed:

{% highlight java %}
// called when the close button has been clicked
windowManager.removeView(rootLayout)
service.stopSelf();
{% endhighlight %}

At the end I put this simple service in a library and have created a facade class called `DebugOverlay` as public API. This class is also responsible to start and bind the service and to submit new log message to this service. So we can use the `DebugOverlay` like this:

{% highlight java %}
DebugOverlay.with(context).log("Hello World");
{% endhighlight %}

Next you may ask how do I ensure that this debug overlay window is just added to debug builds and not to the release builds. I have created a second library artifact with the same `DebugOverlay` facade API but does nothing on method calls.

{% highlight groovy %}
debugCompile('com.hannesdorfmann:debugoverlay:0.2.0') // Starts the service and displays the overlay
releaseCompile('com.hannesdorfmann:debugoverlay-noop:0.2.0') // Does nothing
{% endhighlight %}

I use gralde build types to include the `debugoverlay` dependency that do display the debug overlay window in debug builds and use the other `debugoverlay-noop` dependency for release builds.

The `debugoverlay-noop` adds 1 class and 3 methods to your release dex file, which from my point of view is an acceptable compromise.

**Please note** that `com.hannesdorfmann:debugoverlay:0.2.0` will add `android.permission.SYSTEM_ALERT_WINDOW` permission to your `apk`. Hence you should avoid to use that dependency in your release `.apk`

You can find this tiny library on [github](https://github.com/sockeqwe/debugoverlay) and is available on maven central.

### Update
It seems that I have missed some already existing libraries that coulddo the same job as `DebugOverlay`:

  - [GhostLog](https://github.com/jgilfelt/GhostLog) by Jeff Gilfelt
  - [Galgo](https://github.com/inaka/galgo) by Inaka
  - [Lynx](https://github.com/pedrovgs/Lynx) by Pedro Vicente Gómez Sánchez
