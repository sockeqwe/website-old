---
layout: post
published: true
title: FragmentArgs 3.0
mathjax: false
featured: false
comments: true
headline: FragmentArgs 3.0
categories:
  - Android
tags: [android, annotation-processing]
---
I finally found some time last weekend to update [FragmentArgs](https://github.com/sockeqwe/fragmentargs) and to release a new major version **3.0**. Here is an overview of new features and bug fixes.

## Migration
The good news first: FragmentArgs 3.0 is completely backward compatible to all previous versions (down to `1.0`). So basically you have to do nothing. However, I recommend to add the `@FragmentWithArgs` annotations to your Fragments that contain `@Arg` annotations. Why? See _What's new_ section and _Bugfixes_ sections.

## What's new?
 - I have added `@FragmentWithArgs` and you should annotate all Fragment classes with this annotation. For backward compatibility reasons `@FragmentWithArgs` is not mandatory. However, I strongly recommend to use `@FragmentWithArgs` because in further versions of FragmentArgs this could become mandatory to support more features.
 - `@FragmentArgsInherited` is deprecated now. Use `@FragmentWithArgs(inherited = true or false)` instead.
 - The generated Builder classes are now annotated with `@NonNull` from android support annotations library. This allows LINT to check your builders as well. You can disable that (see _"Annotation Processor Options"_ below)
 - Setter methods: You still have to annotate fields with `@Arg` (not the setter method itself). FragmentArgs detects the corresponding setter method automatically. This allows you to annotate private or protected fields with `@Arg`. However, I still recommend to use package (default) visibility for your argument fields:

{% highlight java %}
@FragmentWithArgs
public class MyFragment extends Fragment {

    @Arg
    int id;

    @Arg
    private String mTitle; // private fields requires a setter method

  // Setter method for private field.
  // The name must be the "set" prefix concatenated with field name
  public void setTitle(String title){
    this.title = title;
  }

}
{% endhighlight %}

FragmentArgs supports hungarian notation for detecting fields (even if your setter method would be called `setMTitle()` or `setmTitle()`). Nevertheless, I'm not a fan of the hungarian notation. We are not old school C++ developers living in the 1990's. We should stop using the hungarian notation on android.

## Kotlin support
The kotlin programming language is supported (use `kapt` instead of `apt`). Since FragmentArgs 3.0 now supports setter methods kotlin backing fields are supported out of the box (because backing fields simply generate getter and setters).

{% highlight kotlin %}
 @FragmentWithArgs
 class KotlinFragment : Fragment() {

     @Arg var foo: String = "foo"
     @Arg(required = false) lateinit var bar: String // works also with lateinit for non primitives

     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)
         FragmentArgs.inject(this)
     }

     override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
         val view = inflater.inflate(R.layout.fragment_kotlin, container, false)

         val tv = view.findViewById(R.id.textView) as TextView

         tv.text = "Foo = ${foo} , bar = ${bar}"
         return view;
     }
 }
 {% endhighlight %}

## Annotation Processor Options
The FragmentArgs annotation processor supports some options for customization.

{% highlight groovy %}
// Hugo Visser's APT plugin
apt {
  arguments {
    fragmentArgsLib true
    fragmentArgsSupportAnnotations false
    fragmentArgsBuilderAnnotations "hugo.weaving.DebugLog com.foo.OtherAnnotation"
  }
}

// Kotlin Annotation processor
kapt {
  generateStubs = true
  arguments {
    arg("fragmentArgsLib", true)
    arg("fragmentArgsSupportAnnotations", false)
    arg("fragmentArgsBuilderAnnotations", "hugo.weaving.DebugLog com.foo.OtherAnnotation")
  }
}
 {% endhighlight %}

 - **fragmentArgsLib**: If you want to use FragmentArgs in android library projects. See [github readme](https://github.com/sockeqwe/fragmentargs) for more informtation.
 - **fragmentArgsSupportAnnotations**: As default the methods of the generated `Builder` are annotated with the annotations from support library like `@NonNull` etc. You can disable that feature by passing `false`.
 - **fragmentArgsBuilderAnnotations**: You can add additional annotations to the generated `Builder` classes. For example you can add `@DebugLog` annotation to the `Builder` classes to use Jake Wharton's [Hugo](https://github.com/JakeWharton/hugo) for logging in debug builds. You have to pass a string of a full qualified annotation class name. You can supply multiple annotations by using a white space between each one. The sample show above would generate a builder class like this:

 {% highlight java %}
  @hugo.weaving.DebugLog
  @com.foo.OtherAnnotation
  public final class MyFragmentBuilder {
    ...
  }
 {% endhighlight %}

## Bugfixes:
 -  [#30](https://github.com/sockeqwe/fragmentargs/issues/30): There was a bug when having a Fragment with only optional arguments `@Arg(required = false)`. This has been fixed.
 - [#32](https://github.com/sockeqwe/fragmentargs/issues/32): It wasn't possible to generate a builder for a Fragment without any `@Arg` annotation.  **TL;DR**: Use `@FragmentWithArgs` also on Fragments with no arguments and the corresponding generated `Builder` to instantiate an instance.
 The long version: So why would should you use the FragmentArgs Builder if you don't have arguments anyway? Imagine you have a Fragment called `MyFragment`. If it doesn't have any argument then it's safe to instantiate the Fragment directly via `new MyFragment()`. However, you should always use a FragmentArgs builder: Why? The reason is consistency and compile time verification. Imagine that one day in the future `MyFragment` needs an argument. So now you would have to scan all of your code for `new MyFragment()` calls and replace them with the `new MyFragmentBuilder(argument).build()`. What if you overlook one `new MyFragment()` statement. So your code will still compile, but will crash at runtime because the argument is missing. If you would have used `new MyFragmentBuilder()` instead even for fragments without arguments from the beginning you won't run into this kind of issue because as soon as you add an `argument` by using `@Arg` to `MyFragment` class the Builder will throw an compile error because the argument is required as constructor parameter in `new MyFragmentBuilder(argument)`.
