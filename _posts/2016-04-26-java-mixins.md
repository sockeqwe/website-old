---
layout: post
published: true
title: Mixins in Java
mathjax: false
featured: false
comments: true
headline: Mixins in Java
categories:
  - Android
tags: [android, java, design-patterns]
---

Starting in Android N Google has added some java 8 language features. One of those features is the ability to add default methods to interfaces. Surprisingly (since java 8 has already been released 2 years ago) I haven't found good articles describing the advantage of default methods for interfaces: **Mixins!**

Let's straight jump into a simple example. Let's say we have a class `Ship`. A `Ship` can carry `Cargo`. Let's model that in Java like this:

```java
public class Ship {

  List<Cargo> cargoes;

  public void addCargo(Cargo c){
    cargoes.add(c);
  }

  public void removeCargo(Cargo c){
    cargoes.remove(c);
  }
}
```

We also have an `Airport` where `Aircrafts` can land and depart:

```java
public class Airport {

  List<Aircraft> aircrafts;

  public void land(Aircraft a) {
    aircrafts.add(a);
  }

  public void depart(Aircraft a) {
    aircrafts.remove(a);
  }
}
```

Given that, what if we want to have another class that is `Airport` and `Ship` at the same time? Unrealistic? What about an `AircraftCarrier`. Since java doesn't support multiple inheritance we can't declare a class `AircraftCarrier` that extends from `Airport` and `Ship`:

```java
class AircraftCarrier extends Airport, Ship // doesn't compile
```

So what alternatives do we have to make `AircraftCarrier` be a `Ship` and a `Airport` at the same time? Well, we could use delegation ("favor composition over inheritance") or some tricks from  [aspect](https://javadeveloperslife.wordpress.com/2013/06/17/Mixins-with-pure-java/)  orientated programming, but none of them are really natively supported by java programming language.

However, java does support multiple inheritance on interfaces and allows classes to implement arbitrary many interfaces. So let's convert `Ship` and `Airport` to interfaces:

```java
public interface Ship {
  void addCargo(Cargo c);
  void removeCargo(Cargo c);
}

public interface Airport {
  void land(Aircraft a);
  void depart(Aircraft a);
}

class AircraftCarrier implements Ship, Airport {

  List<Aircraft> aircrafts;
  List<Cargo> cargoes;

  public void land(Aircraft a) {
    aircrafts.add(a);
  }

  public void depart(Aircraft a) {
    aircrafts.remove(a);
  }

  public void addCargo(Cargo c){
    cargoes.add(c);
  }

  public void removeCargo(Cargo c){
    cargoes.remove(c);
  }
}
```

But now we have to implement the methods from each interface manually. Not a big deal in this simple example because it's basically just one line of code for each method but I guess you get the point.

How does Ruby programmers deal with inheritance? Experienced ruby programmers don't use inheritance for such things. They use Mixins:

```ruby
module Airport  # for simplicity: module is same as class

  @aircrafts = Array.new    # variable

  def land(aircraft)
    aircrafts.push(aircraft)
  end

  def depart(aircraft)
    aircraft.delete(aircraft)
  end
end


class AircraftCarrier
  include Ship
  include Airport
end
```

With Mixins we can compose `AircraftCarrier` by "including" the functionality of Ship and Airport without inheritance. Ship definition has been omitted for better readability but I guess you can imagine how the code for `module Ship` will look like. Please note that this is not multiple inheritance as C++ offers. Mixins are a different concept. The Ruby programming language has native support for Mixins. What about other languages like Scala? Scala has support for Mixins as well. In Scala those are called Traits. Traits are Mixins just with some slightly different properties from a programming language designers point of view like Mixins require linearization while Traits are flattened and Traits traditionally don't contain states. But that shouldn't worry you too much. For the sake of simplicity we can say Mixins and Traits are the same.

```scala
trait Ship {
  val cargoes : ListBuffer[Cargo]

  def addCargo(c : Cargo){
    cargoes += c
  }

  def removeCargo(c : Cargo){
    cargoes -= c
  }
}
```

And then with Scala we can do something like this:

```scala
class AircraftCarrier with Ship with Airport
```

See the thing is both, Ruby and Scala, natively have language support for Mixins. But what about java? With Java 8 and default methods for interfaces we can do pretty the same thing:

```java
public interface Airport {

  // To be implemented in subclass
  List<Aircraft> getAircrafts();

  void land(Aircraft aircraft) {
    getAircrafts().add(aircraft);
  }

  void depart(Aircraft aircraft) {
    getAircrafts.remove(aircraft);
  }
}
```

We will do the same for `Ship` (interface with default method implementations) and then we can do something like this:

```java
class AircraftCarrier implements Ship, Airport {

  List<Aircraft> aircrafts = new ArrayList<>();
  List<Cargo> cargoes = new ArrayList<>();

  @Override
  public List<Aircraft> getAircrafts(){
    return aircrafts;
  }

  @Override
  public List<Cargo> getCargoes(){
    return cargoes;
  }
}
```

```java
AircraftCarrier carrier = new AircraftCarrier();
carrier.addCargo(c);
carrier.land(a);
```

Moreover, now we can compose new classes even more easily and without traditional inheritance:

```java
class Houseboat implements House, Ship { ... }

class MilitaryHouseboat implements House, Ship, Airport { ... }
```

I guess you get the point. With java 8 and interfaces default methods we can use Mixins instead of inheritance. kotlin also offers Mixins by interfaces with default methods:

```scala
interface Ship {
  val cargoes : List<Cargo>

  fun addCargo(c : Cargo){
    cargoes.add(c)
  }

  fun removeCargo(c : Cargo) {
    cargoes.remove(c)
  }
}

class AircraftCarrier : Ship, Airport {
  override val cargoes = ArrayList()
  override val aircrafts = ArrayList()
}
```

I assume you get the point why Mixins may be a better choice than traditional inheritance. But how does that help in android development. In android we always have to extend from Activity or Fragment, which are obviously two different base classes. With Mixins, we can share code between both classes that extends from Activity and classes that extends from Fragment. Sure this is not the main use case of Mixins (delegation might be a better choice) because those methods defined in the Mixin will be used internally from the Fragment or Activity himself which are implementing the Mixin interface, but it is a valid option though.

See the moral of the story is it is good to have an alternative to inheritance and I have the feeling that unfortunately Mixins via interfaces with default implementation are not used that much in the java world.

My general advice about inheritance:

> inheritance doesn't scale well! Use inheritance for inheriting properties but not for inheriting functionality!
