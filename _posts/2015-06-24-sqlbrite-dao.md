---
layout: post
published: true
title: SQLBrite DAO
mathjax: false
featured: false
comments: true
headline: SQLBrite DAO
categories:
  - Android
tags: [android, software-architecture, annotation-processing]
---

Earlier this year, one day before Valentine's day to be precise, I had the glorious idea _(ironie)_ to surprise my lovely girlfriend with a special android app on Valentine's day. Who said computer science can't be romantic?

The idea was simple: I wanted to build an android app that imitates and looks like [Tinder](https://play.google.com/store/apps/details?id=com.tinder&hl=de) but instead of displaying real people nearby, I faked everything so that only my profile gets displayed so she only can choose me. Nearly everything was hard coded except the messaging part. To save time and effort I didn't want to build my own backend to store messages and provide a REST api. Instead I decided to use GCM to send and deliver chat messages form my girlfriend's phone to my phone and vice versa. That required to store the received chat messages on a local SQLite database on the users device. Therefore a database library was needed.

#Database
To cut a long story short - I didn't managed to implement that fake Tinder app in time. One of the reasons was that I hadn't found a good and simple database abstraction layer library. I could pick just a popular one to get the job done, but that wasn't what I wanted to do. Moreover, I was looking for a library that matched the following criteria:

 - **Native SQL:** I don't like ORM libraries because usually they require to learn their own query language and declarative table schemas. To build efficient queries I have to learn how to use that ORM library (I look at you Hibernate!). If I change ORM library one day, I have to learn how to write efficient queries again for the new ORM library. Moreover, ORM libraries has their own implementations how to save and resolve relations. Sometimes, you have to adjust your model (pojo) classes to make efficient queries possible just because of an ORM implementation detail. I'm very familiar with SQL which is universal useable (obviously also outside of the android world). I already know how to build efficient queries in SQL. Therefore, I want to write my queries in pure native SQL. Finally, I came to the conclusion that hiding SQL is not the best idea.
 - **Based on RxJava:** Many developers are excited about Rx programming, because Rx programming offers functional alike operators like `flatMap()` etc. Many of them don't understand that by using RxJava they are observing data. Therefore, they don't understand Rx programming at all. It is not only about transforming data. Rx implements the observer pattern. You are subscribing to an observable to get updates. That may be a one time consumable data stream like a http response, but the real power of the observer pattern and RxJava can be seen and used by making a data source observable that can emit more than once items, like a database does: Whenever the data in your database has been changed, all subscribers can be informed about the change by emiting items (so that `onNext()` gets called again).
 - **Immutability:** Almost every ORM based library misses that point. I'm not going to explain the advantages and disadvantages of immutability. I recommend to read [Effective Java](http://www.amazon.com/exec/obidos/ASIN/0321356683/ref=nosim/javapractices-20) by Joshua Bloch or have a look at [javapracties.com](http://www.javapractices.com/topic/TopicAction.do?Id=29) which sums up the most important things.

 I know that this may sounds like an overkill for such a simple app like the fake Tinder app is. It just stores chat messages into a database. Whenever I build an app, I want to ensure that it's build the best way and even more important, with every new app (even with such a little app like the fake Tinder app) I improve my skills and learn new things.

#SQLBrite
Don't finishing the app in time also had its advantages: A few days after Valentine's day the guys over at square (who else?) open sourced [SQLBrite](https://github.com/square/sqlbrite) which seems to be the database layer I am dreaming of: It's a lightweight wrapper around `SQLiteOpenHelper` and `ContentResolver`, it doesn't hide SQLite or SQL API, supports transactions and multithreading and last but not least it introduces reactive stream semantics to queries. Especially, the latter one is worth mentioning: Whenever a row of a sql database table gets updated, inserted or removed SQLBrite triggers a notification to inform queries, which are subscribed for table dataset changes. So I decided to use SQLBrite in the fake Tinder app.

In this fake Tinder app you can open `ChatActivity` which displays a `List<ChatMessage>` queried from the local SQLite database. As already mentioned before I use GCM to send and receive chat messages. Whenever the app receives a GCM Push notification containing a chat message, the app stores the `ChatMessage` into the local database. The big advantage of using SQLBrite is that by inserting a new `ChatMessage` into the local database `ChatActivity` gets updated automatically because as long as `ChatActivity` is not destroyed the original query executed in `ChatActivity.onCreate()` to retrieve `List<ChatMessage>` is still subscribed (query is  Rx Subscriber) to the underlying database table (database table is Rx Observable). But the really cool thing is that you get that update mechanism for free. You don't have to add a single line of code. SQLBrite and RxJava "magically" do that. No `EventBus` to start a re-query manually, pure reactive programming power. So whenever the app receives a GCM push notification containing a `ChatMessage` it stores this `ChatMessage` into the local SQLite database. If `ChatActivity` is open while receiving the GCM push notification SQLBrite will automatically deliver the database changes to `ChatActivity`. It's that easy to keep `ChatActivity` up to date. This Activity don't even have a pull-to-refresh mechanism like a `SwipeRefreshLayout` because updates are pushed from SQLBrite automatically. Therefore, there is no reason to implement a pull mechanism.

#SQLBrite Dao
SQLBrite is still in it's early days (Version 0.1.0 while writing this blog post). As already said the main focus of SQLBrite is set on providing a wrapper arround `SQLite`. No ORM and no type-safe query mechanism are provided. So by using SQLBrite you have to work with `Cursor` and `ContentValues`. That was a little bit annoying while developing the fake Tinder app. Therefore, I decided to write an [annotation processor](hannesdorfmann.com/annotation-processing/annotationprocessing101/) for "very simple" object mapping and [DAO](https://en.wikipedia.org/wiki/Data_access_object) (Data Access Object) on top of SQLBrite. SQLBrite Dao can be found on [Github](https://github.com/sockeqwe/sqlbrite-dao).

###Object mapping
Please note that this is not an ORM. The only thing it does is take a `Cursor` and read the column into the model class pojo. Only primitives are allowed, no relations like 1:n (1 `Chat` has many `ChatMessages`) can't be modeled and resolved. It's more like deserializing data than object mapping. You have to annotate your model class with `@ObjectMappable` and the desired fields with `@Column` with the table column name (String) as parameter:

{% highlight java %}
@ObjectMappable
public class ChatMessage {

  //
  // Define Table name and column names as constants
  //
  public static final String TABLE_NAME = "ChatMessage";
  public static final String COL_ID = "id";
  public static final String COL_SENDER_NAME = "sender";
  public static final String COL_MESSAGE = "lastname";
  public static final String COL_TIMESTAMP = "received"
  public static final String COL_CHAT_ID = "chat_id"

  //
  // Fields mapped to database columns
  //
  @Column(COL_ID)
  String id;

  @Column(COL_SENDER_NAME)
  String sender;

  @Column(COL_MESSAGE)
  String message;

  @Column(COL_TIMESTAMP)
  long timestamp;

  @Column(COL_CHAT_ID)
  String chatId;

  public ChatMessage() {
  }

  public String getId(){
    return id;
  }

  public String getSender(){
    return sender;
  }

  public String getMessage(){
    return message;
  }

  public long getTimestamp(){
    return timestamp;
  }

  public String getChatId(){
    return chatId;
  }
}
{% endhighlight %}

An annotation processor (not reflections) then generates `ChatMessageMapper` class. That generated class (original `@ObjectMappable` annotated class name + "Mapper" suffix) looks like this:

{% highlight java %}
public final class ChatMessageMapper {
  private ChatMessageMapper() {
  }

  /**
   * Retrieves the first element from  Cursor by scanning for @Column annotated fields.
   * @param cursor The Cursor
   * @return null if Cursor is empty or a single ChatMessage fetched from the Cursor
   */
  public static ChatMessage single(Cursor cursor) {
      ...
  }

  /**
   * Fetches a list of ChatMessage from a Cursor by scanning for @Column annotated fields.
   * @param cursor The Cursor
   * @return An empty List if cursor is empty or a list of ChatMessage items fetched from the Cursor
   */
  public static List<Customer> list(Cursor cursor) {
    ...
  }


  /**
   * Get a type-safe ContentValues Builder
   * @return The ContentValues Builder
   */
  public static ContentValuesBuilder contentValues() {
    ...
  }

}
{% endhighlight %}

So you can call `ChatMessage msg = ChatMessageMapper.single(Cursor)` or `List<ChatMessage> messages = ChatMessageMapper.list(Cursor)` to read the Cursor and save the cursor column values into the correspondig fields of the model class. You may have also noticed that there is a `ContentValuesBuilder` that can be used as type-safe builder for `ContentValues` like this:

{% highlight java %}
ContentValues cv = ChatMessageMapper.contentValues()
                        .id("1")
                        .sender("Hannes")
                        .message("Will you be my Valentine?")
                        .timestamp(1108389600)
                        .chatId("chat1")
                        .build();
{% endhighlight %}

It's just a helper class that makes working with `Cursor` or `ContentValues` more convenient.

###DAO
Create your own **D**ata **A**ccess **O**bject  (DAO) where you define methods to manipulate or query your database table. `Dao` provides SQL grammar so you don't have to deal that much with String concatenation and can use IDE's auto completion to build your sql statements. Usually a DAO represents a database table as following:

{% highlight java %}
public class ChatMessageDao extends Dao {

  @Override public void createTable(SQLiteDatabase database) {

    CREATE_TABLE(ChatMessage.TABLE_NAME,
        ChatMessage.COL_ID + " TEXT PRIMARY KEY NOT NULL",
        ChatMessage.COL_SENDER_NAME + " TEXT",
        ChatMessage.COL_MESSAGE + " TEXT",
        ChatMessage.COL_TIMESTAMP + " INTEGER",
        ChatMessage.COL_CHAT_ID+" TEXT"
      )
        .execute(database);

  }

  @Override public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
    if (oldVersion == 1 && newVersion == 2){
      ALTER_TABLE(ChatMessage.TABLE_NAME)
          .ADD_COLUMN(ChatMessage.COL_FOO +" TEXT")
          .execute(db);
    }
  }


  public Observable<List<ChatMessage>> getMessages(String chatId) {
        return query(

        SELECT(
            ChatMessage.COL_ID,
            ChatMessage.COL_SENDER_NAME,
            ChatMessage.COL_MESSAGE,
            ChatMessage.COL_TIMESTAMP
            )
          .FROM(ChatMessage.TABLE_NAME)
          .WHERE(ChatMessage.COL_CHAT_ID + " = ? ") ,

          chatId // Argument that replaces "?" in WHERE
        )
        .map(new Func1<SqlBrite.Query, List<ChatMessage>>() {  // SqlBrite.Query to List<ChatMessage>
          @Override public List<ChatMessage> call(SqlBrite.Query query) {
            Cursor cursor = query.run();
            return ChatMessageMapper.list(cursor);  // Generated Mapper class
          }
        });
      }


  public Observable<Long> addMessage(String id, String sender, String msg, long ts, String chatId ) {
    ContentValues values = ChatMessageMapper.contentValues()
                         .id(id)
                         .sender(sender)
                         .message(msg)
                         .timestamp(ts)
                         .chatId(chatId)
                         .build();

    return insert(ChatMessage.TABLE_NAME, values);
  }

}
{% endhighlight %}

To register your `Dao` classes to `SQLBrite` you have to create a `DaoManager`. While a `Dao` represents a table of a database `DaoManager` represents the whole database file. `DaoManager` internally creates a `SQLiteOpenHelper` and instantiates a `SqlBrite` instance. All DAO's registered to the same `DaoManager` share the same `SqlBrite` instance.

{% highlight java %}
ChatMessageDao cmDao = new ChatMessageDao();
ChatDao chatDao = new ChatDao();

int dbVersion = 1;

DaoManager daoManager = new DaoManager(context, "Chat.db", dbVersion, cmDao, chatDao);
daoManager.setLogging(true);
{% endhighlight %}

Please note that adding DAO's dynamically (later) is not possible. You have to instantiate a `DaoManager` and pass all your DAO's in the constructor as seen above. DaoManager calls `Dao.createTable()` and `Dao.onUpgrade()` once the internal `SQLiteOpenHelper` is opened.

#Summary
SQLBrite Dao provides an additional layer on top of SQLBrite:

 - A `DaoManager` is representing the whole database file and basically is a `SQLiteOpenHelper` and manages `SqlBrite` instance for you.
 - A `Dao` is representing a table of a database. You define a public API for other software components of your App like `getMessages()` or `addMessage()` to query and manipulate the data of the underlying table.


 I have build this "DAO" and "ObjectMapper" to make things easier while working with SQLBrite by providing a little bit more high level APIs around SQLBrite. As already said SQLBrite is still under development and I think (and hope) that some of this things like "simple" object mapping or SQL grammar will be part of `SQLBrite 1.0`. So why do I have built this additional library? Working with `Cursor` and  `ContentValues` was annoying and also writing SQL statements by using String concatenation isn't really ahead of time. I really like SQLBrite but until now (end of June 2015) such things are not provided (yet) by SQLBrite. While my DAO and ObjectMapper are working and can be used in real world apps I would recommend to use it in your hobby projects or if you just want to play around with SQLBrite by writing a demo app. The annotation processor based `ObjectMapper` should definitely be improved to something like [AutoValue](https://github.com/google/auto/tree/master/value) (I'm a big fan of AutoValue) to support real Immutability. You may ask why I didn't do that from the very beginning: Well many parts of annotation processor code are based on another database library I have written some years ago. Same is true for DAO and SQL grammar which were refactored out of other libraries and apps of mine. Before I put more effort into this library I wanted to check which features are planned to be part of SQLBrite 1.0. Right now (end of June 2015) square is working on version 0.2.0 which as far as I see don't already provide such features but rather is focusing on the core functionality: working with SQLite databases which per se is already a tough topic. That's also the reason why I have decided to open source [SQLBrite Dao](https://github.com/sockeqwe/sqlbrite-dao) and to write this blog post about it to explain the idea behind it. SQLBrite Dao is not an official part of SQLBrite. To repeat it: It is more for those people who are currently working on hobby projects or playing around with SQLBrite a little bit. Those guys may find this library useful. Hence I thought it wouldn't hurt to open source it.

 SQLBrite Dao can be found on [Github](https://github.com/sockeqwe/sqlbrite-dao)

To finish the story about the fake Tinder app: As already said I haven't made it to implement this app in time, but `SQLBrite Dao` was one of the out coming artifacts. Actually, this happens quite often to me: I start a new app project and end up writing a library that would be nice to have for this already started app project. Usually, I end up with having build a library and the app itself never gets finished. However, this time I finished the app itself (the fake Tinder app) as well, unfortunately after Valentine's day. Therefore, I had to go the classic way for Valentine's day 2015 and I bought my girlfriend red roses and went to romantic dinner with her. I can't wait for Valentine's day next year when I will install the fake Tinder app over night on her smartphone and wait for "having a match" on Valentine's day.

P.S: Don't worry, my girlfriend is not a computer scientist nor internet addicted (that's why I love her). Actually, she don't even know that I have a blog. As long as you don't tell her directly about the fake Tinder app, it will still be a surprise on Valentine's day 2016. Keep your lips sealed ;-)
