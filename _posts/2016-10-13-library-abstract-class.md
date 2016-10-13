---
layout: post
published: true
title: Why a library developer may should consider using abstract class instead of interface
mathjax: false
featured: false
comments: true
headline: Why a library developer may should consider using abstract class instead of interface
categories:
  - Android
tags: [android, java]
---

Use interfaces for java development they said. It will be more flexible they said. Well, that all might be true, but for library projects that doesn't necessarily has to be true as well. In this post I will explain you why I have switched from `interface` to an `abstract class` as base class in one of my library projects called [AdapterDelegates 3.0](https://github.com/sockeqwe/AdapterDelegates).

To give you some background information: AdapterDelegates is a small library I have written to create composable Adapters for Android's RecyclerView (favor composition over inheritance). I have described the idea and the reason why we need such a library in a previous blog post: [Joe's great adapter hell escape](http://hannesdorfmann.com/android/adapter-delegates).

Basically, AdapterDelegates (before version 3.0) had an interface called `AdapterDelegate` like this:

```java
/**
 * @param <T> the type of adapters data source i.e. List<Foo>
 */
public interface AdapterDelegate<T> {

  /**
   * Called to determine whether this AdapterDelegate is the responsible for the given data
   * element.
   *
   * @param items The data source of the Adapter
   * @param position The position in the datasource
   * @return true, if this item is responsible,  otherwise false
   */
  public boolean isForViewType(T items, int position);

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
  public void onBindViewHolder(T items, int position, RecyclerView.ViewHolder holder);
}
```

So for every view type you want to display in a `RecyclerView` you had to define your own class implementing `AdapterDelegate` interface and then you could plug-in multiple  AdapterDelegates into a RecyclerView's Adapter to display different items (view types). Moreover, you could reuse the same AdapterDelegate for multiple Adapters.

## class vs. interface
One may ask: Why have you defined `interface AdapterDelegate<T>` and not simply `class AdapterDelegate<T>`? Well, an AdapterDelegate is just a contract (or a protocol) that says this methods have to be implemented. That is exactly what interfaces are good for. Furthermore, we should program against interfaces, right? Instead of writing your classes in a way that says _"I depend on this specific class to do my work"_  with interfaces it's more like _"I depend on any class that does this stuff to do my work"._ By doing so we don't rely on implementation details, are more flexible and loosely coupled.

## interface vs. abstract class
That is a little bit more tricky although the same argument as in the _"class vs. interface"_ paragraph from above is still valid. The main difference is that abstract classes let you define some behaviors and they force your subclasses to provide others (abstract methods). But that is also a problem. Inheritance (in contrast to implementing an interface) might introduce a shared state and behavior relation between super class and subclass because the state of the overall object relies on implementation of the super class and subclass. Moreover, that means that if your are extending from a abstract class and you are not implementing the abstract methods as intended by the author of the super class (which might not be you) you may break internal state and behavior of your subclass. Read this sentence again. What I'm trying to say is: if you don't know all the implementation details of your super class, you can't be sure that your subclass is working correctly. I'm pretty sure you have extended from abstract classes and have implemented the missing abstract methods before but have you ever checked the source code of the super class to be sure that your implementation of those abstract methods are as intended by the author of the super class? By the way the source code of a super class  could have been changed with every update. You might have to look at the source code again to verify that your subclass implementation still conforms with the intention of the super class.

## Abstract classes allow default implementations
With that said, you are wondering why I have switched from `interface AdapterDelegate<T>` to `abstract class AdapterDelegate<T>` in AdapterDelegates version 3.0, aren't you? For a library `abstract classes` do make sense **if your libraries public API changes frequently** and if you are not under full control of how and when the API will be changed.

That is exactly the case for my AdapterDelegates library. I depend on the RecyclerView's Adapter API which, obviously, is not designed nor maintained by me. Concrete example:
Few releases back a new method `onBindViewHolder(VH holder, int position, List<Object> payloads)` has been added to RecyclerView.Adapter class to support payloads. AdapterDelegates 2.0 interface only contained the method signature `onBindViewHolder(VH holder, int position)` (without payloads).  If I want to add this method to `interface AdapterDelegate<T>` in version 2.1 everybody using my library would have to go into his source code and implement this method too. Otherwise his / her code wouldn't compile.

What if you decide to update to version 2.1 (with payload support) but a third party dependency of your app still depends on version 2.0 (without payload support)?

![dependencies](/images/adapterdelegates/dependencies.png)

Then your code will compile but your app will crash at runtime. Why? Because the third party library is already compiled. Hence, no compile time error will be thrown but gradle will pack version 2.1 (can't pack both 2.1 and 2.0 in the same apk, therefore uses the newer one) into your final android APK file. Then when invoking `onBindViewHolder(VH holder, int position, List<Object> payloads)` on a third party library component a `NoSuchMethodError` will be thrown at runtime. The android SDK faces the same issue. LINT warns you to add a check like `if (VERSION.SDK_INT >= 21)` but there is no such mechanism for libraries packed in your code.

To avoid such problems Jake Wharton suggested to change package name and maven group id in his blog post [Java Interoperability Policy for Major Version Updates](http://jakewharton.com/java-interoperability-policy-for-major-version-updates/).
That is a very good strategy you should follow when publishing your own library. But what is a major version update? In my case, every time RecyclerView's Adapter changes his API that would be a major version update for my AdapterDelegates library too because I may have to add new methods to the `interface AdapterDelegate<T>`. That wouldn't be very inconvenient for the users of my library.

Therefore, I have decided to switch to `abstract class AdapterDelegate<T>` because most likely the development team behind RecyclerView will add new methods that bring new optional features. At least that was the case in the past. By using abstract class instead of an interface I can add this methods "silently" in a minor version update (not major version update) by providing a default implementation, which abstract classes allow me to do, but interfaces don't (before java 8).

Now you may roll your eyes and ask: What about all that inheritance shared state and behavior nonsense I told you before and that you can't be sure that by inheriting from a super class you aren't breaking anything without checking super class source code.

Well, that still is true. However, I as a library developer restrict myself to define `abstract class AdapterDelegate<T>` just like I would define an interface by using abstract methods except the fact that for newer optional features I will provide an empty default implementation:

```java
public abstract class AdapterDelegate<T> {

  protected abstract boolean isForViewType(T items, int position);

  protected abstract RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent);

  protected abstract void onBindViewHolder(T items, int position, RecyclerView.ViewHolder holder, List<Object> payloads);

  protected boolean onFailedToRecycleView(RecyclerView.ViewHolder holder) {
    return false;
  }

  protected void onViewAttachedToWindow(@NonNull RecyclerView.ViewHolder holder) {
  }

  protected void onViewDetachedFromWindow(RecyclerView.ViewHolder holder) {
  }
}
```

So you, as user of this library, can checkout the source code and see that there is no shared state or behavior with your own subclass.


TL;DR: For a library project it is okay to use abstract class instead of interface, if you restrict yourself to design abstract classes like interfaces and only add empty default implementations but no default implementation that changes state or behavior of this class (and in consequence of all subclasses).
