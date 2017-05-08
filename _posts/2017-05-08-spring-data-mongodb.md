---
title:  "Spring Data MongoDB - use case study"
date:   2017-05-08 22:00:00 +0100
categories: programming
excerpt: Learn when it's a good idea to make use of NoSQL database and how it's done with Spring Data.
---

Last time I've promised you to write about database persistance and I feel relieved to bring this promise to life with this post. I've been quite busy introducing new features to the HTML editor. I'm perfectly aware of the fact that this can be a neverending story, but each one seems indispensable... Slowly, I'm getting sick of frontend development.

Fortunately, P1 was in need of a solution, which will allow for persistance of edited HTML code in a database. That's a perfect occasion to get hands dirty with Spring Data project. It's really, really awesome. If you haven't used it before, but on the other hand you've had a "pleasure" to work with e.g. JDBC, you'll literally fall in love with how intuitive and straightforward it is. That's how it was in my case, at least.

I've just mentioned JDBC, but actually I won't handle relational databases. No JPA, no Hibernate, no Spring Data JPA. I'll take a look at Spring Data MongoDB. MongoDB is an open-source, NoSQL database. Data isn't stored in tables, but in documents, which, in case of MongoDB, could be simply represented with JSON. NoSQL is complete opposite of SQL - especially it could be described as denormalized, which means that data isn't coupled (or at least it isn't so tightly coupled as with _relational_ - mark the word - databases, with the use of keys and so on). This design implicates some crazy benefits:
* there's no need for schema definition (because there's no schema at all, baby!), which means you could start your project in the blink of an eye plus introducing any modifications is harmless
* thanks to the denormalization, it's much, much more easier to achieve better performance in operations (e.g. no need for executing JOINs to retrieve data) - but keep in mind, that well-designed relational database could perform equally well, or even outperform, (badly) designed non-relational database (which doesn't change the fact, that this is a work that has to be done, so the time is consumed and the money is spent)
Above, short description, should give you a basic understanding of "what's what" in NoSQL. If you're looking for more, and especially you're interested in comparison with traditional, SQL database, I encourage you to familiarize yourself with this [article](https://www.sitepoint.com/sql-vs-nosql-differences/).

What's really important is to answer yourself, if your project is well-suited for NoSQL database, as it isn't a "cure for everything". The simplest rule of thumb would be __apply, when pieces of data are unrelated with each other__. That's exactly why I've choosen NoSQL: I just have to persist some HTML codes in database. On the other hand, a classical example of a misguided idea is to apply NoSQL for a social network solution, where connections between users are "heart and soul". A very interesting testimonial (with even more interesting comments section) of such case in Diaspora could be read [on this blog](http://www.sarahmei.com/blog/2013/11/11/why-you-should-never-use-mongodb/).

So, is your use case sufficient for NoSQL solution? If the answer is yes, let's start off by installing MongoDB. Detailed instructions for major platforms can be found at [MongoDB's webpage](https://docs.mongodb.com/manual/installation/). Just follow step by step guide. When you're done, fire up MongoDB in terminal. Last line that will be printed out, points out the default port used by MongoDB:

```
NETWORK [thread1] waiting for connections on port 27017
```

When you're done with MongoDB installation, it's time for the fun port - coding. Update your POM file by adding the following dependency for support to Spring Data MongoDB:

{% highlight xml %}
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
{% endhighlight %}

After dependencies update, you're ready to create Mongo configuration (Java) file in your project:

{% highlight java %}
@Configuration
@EnableMongoRepositories("webapp.repositories")
@EnableMongoAuditing
public class MongoConfig extends AbstractMongoConfiguration {
    @Override
    protected String getDatabaseName() {
        return "HtmlGeneratorDB";
    }

    @Override
    public Mongo mongo() throws Exception {
        return new MongoClient();
    }
}
{% endhighlight %}

Although I generally share the opinion that XML based Spring configuration is better suited when it comes to large projects (like corporation-large) - it's simply more convenient to maintain just a few files which group up configuration for a whole app than lookup dozens of Java configs scattered around the project - in my personal projects, which I consider rather trifle, I prefer more Java developer-centric aproach, that is modern Java configuration. Enough said, let's shortly explain the above code snippet. `@Configuration` annotation points out that this Java file is a configuration file, therefore it'll be scanned as a source of application configuration. Next one is `@EnableMongoRepositories` which, as stated, enables Spring repositories for MongoDB and indicates the package that should be scanned for repositories. The last annotation is `@EnableMongoAuditing`. Auditing is a fancy name for keeping track of operations performed on entities (like who and when created or modified them etc.). I'll use one of auditing annotations provided by Spring Data MongoDB to effortlessly record the entity creation time. In my case, there's no need for any other auditing feature, but if you'd like some more, visit [Spring docs](http://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#auditing.annotations) to get to know available options.

Our custom Java class extends `AbstractMongoConfiguration`, which saves us some coding compared to defining beans for components `MongoClient` and `MongoTemplate`, which in this case are created under the hood. You just have to provide database name (if it doesn't exsist in your MongoDB, it'll be created the first time you run the app) and override method for creating aformentioned `MongoClient`. Keep in mind, that it's a configuration for local development environment.

Guess what? You're done with configuration. To confirm, run your app. Terminal, in which MongoDB instance is run, will print out something like that:

```
NETWORK [thread1] connection accepted from 127.0.0.1:51000 #1 (1 connection now open)
```

In your Spring application log, you'll see:

```
2017-05-05 17:32:36.595  INFO 6788 --- [           main] org.mongodb.driver.cluster               : Cluster created with settings {hosts=[127.0.0.1:27017], mode=SINGLE, requiredClusterType=UNKNOWN, serverSelectionTimeout='30000 ms', maxWaitQueueSize=500}
2017-05-05 17:32:36.656  INFO 6788 --- [127.0.0.1:27017] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:1, serverValue:1}] to 127.0.0.1:27017
2017-05-05 17:32:36.657  INFO 6788 --- [127.0.0.1:27017] org.mongodb.driver.cluster               : Monitor thread successfully connected to server with description ServerDescription[a bunch of params here]
```

Next on our to-do list is definition of a Java class that will be mapped to a MongoDB's document:

{% highlight java %}
@Document
public class GeneratedHtml {

    @Id
    private String id;

    private String htmlCode;

    private String generatorName;

    @CreatedDate
    private LocalDateTime createdDate;

    //getters, setters & constructors
}
{% endhighlight %}

As you've probably guessed, `@Document` enables us to persist this class with the use of Spring Data MongoDB (either by utilizing `MongoTemplate` or autogenerated repositories). `@Id` uniquely identifies the document within the database (and is mostly used by the DB itself - however sometimes you'd like to pass it to a method, like `delete`). It's worth pointing out, that it should be of type String or BigInteger, as discussed [in this StackOverflow topic](http://stackoverflow.com/questions/26574409/spring-data-mongodb-generating-ids-error). The last thing, I'd like you to notice, is `@CreatedDate` annotation which allows for aformentioned auditing of creation date. Audit fields are being set either when save or insert operations are performed, which is when whole entity is being persisted. What's interesting though, and could surprise you in turn, is that Spring implementation won't set creation date if you've previously set entity's ID manually - Spring assumes that in this case it's not the very first time entity is persisted, so it's quite logical, that creation date shouldn't be set in such case. Anyway, you could easily overcome such an obstacle by implementing the `Persistable` interface and `isNew()` method, which will be evaluated to distinguish if entity is new or not. It was even the topic of Spring Data MongoDB's [Jira ticket](https://jira.spring.io/browse/DATAMONGO-946).

Last but not least, we have to create a repository:

{% highlight java %}
@Repository
public interface GeneratedHtmlRepository extends MongoRepository<GeneratedHtml, String> {
    //nothing here!
}
{% endhighlight %}

All we have to do is extend the `MongoRepository` class and provide types for generics: the first one is the type annoted with `@Document` and the second one is the type annoted with `@Id`. What is extraordinary is the fact, that by simply running your application as-is, Spring Data provides automatic implementation of [some CRUD operations](http://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/repository/MongoRepository.html), `count()` and `exsists()`. Let's say it one more time: you don't have to implement the interface yourself in separate class. You simply inject your `@Repository` bean to the class, in which you wish to use it's features, and call the methods.

But that's not even the beginning. `find()` methods are very, very customizable. There are keywords (e.g.: `GreaterThan`, `Between`, `Like`, `IsNotNull`). You can also specify by which field entity(ies) will be selected (e.g. `findAllByGeneratorType()`). And on top of that, it's all autoimplemented for you! Isn't that a true magic? No, it's Spring Data - and all you have to know is detailed in [docs](http://docs.spring.io/spring-data/data-mongo/docs/current/reference/html/).

I've only specified three additional methods:

{% highlight java %}
@Repository
public interface GeneratedHtmlRepository extends MongoRepository<GeneratedHtml, String> {

  void deleteAllByGeneratorName(String generatorName);

  List<GeneratedSignature> findAllByGeneratorName(String generatorName, Pageable pageable);

  long countByGeneratorName(String generatorName);

}
{% endhighlight %}

First one and third one should be self-explanatory, but I'd like to show you how to use the second one:

{% highlight java %}
//returns a [page] of [size] entries, sorted in descending order by creation date
final PageRequest pageable = new PageRequest(page, size, Sort.Direction.DESC, "createdDate");
//queries database
List<GeneratedSignature> generatedSignatures = generatedSignatureRepository.findAllByGeneratorName(generatorName, pageable);
{% endhighlight %}

[`PageRequest`](http://docs.spring.io/spring-data/data-commons/docs/1.6.1.RELEASE/api/org/springframework/data/domain/PageRequest.html) implements `Pageable` interface, which allows for sorting and paging of returned entries, which is super useful. Basically, `PageRequest`'s constructor that I've used takes following parameters:
* `page` and `size` which defines the set of returned entities (if page is 0 and size is 5, it returns 5 first entries; if page is 1 and size is 5, it returns second half of first ten entries)
* sort direction, which could be ascending or descending order
* field names (these are multiple parameters, as declared in method's signature with `String... params`) evaluated by sorting

That's all for this post. I hope you've got a nice intuition how to use Spring Data MongoDB in your own project. Till the next time!
