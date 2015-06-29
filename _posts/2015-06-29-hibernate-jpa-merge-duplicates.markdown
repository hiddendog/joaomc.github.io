---
layout: post
title:  "Solving the JPA2/Hibernate duplicate rows after merge issue"
date:   2015-06-29 09:17:00
comments: true
categories: java jpa hibernate merge duplicate rows hhh-3332 hhh-5855
---

# The issue

I have mixed feelings about JPA. It certainly makes database persistence much easier at first. However, when you hit a wall, it's a really huge wall, a wall of cryptic errors and bizarre behaviours. You can walk arout the wall, yeah, but it's a hundred-mile walk at night in the middle of a desert full of mutant coyotes and no water at all. You may have no choice but find out how to make a hole into the fucking wall and go right through it.

Let me tell you about a wall I've hit a couple of times. It's the wall of duplicated database rows after a merge operation. There are Hibernate bugs filed: [HHH-5855](https://hibernate.atlassian.net/browse/HHH-5855) [HHH-3332](https://hibernate.atlassian.net/browse/HHH-3332), and also many people that face [this](http://stackoverflow.com/questions/1930966/jpa-merge-is-causing-duplicates) [problem](http://stackoverflow.com/questions/1975708/entitymanager-merge-inserts-duplicate-entities). I managed to destroy the wall twice and I can give you a couple of advices:

# The advices

## Always use the result of the "merge" operation

The first time I solved the "duplicate rows after merge" problem reassigning the bean after the merge. To understand what this means and why you need to do it, you first need to know what the "merge" operation really does. Remember, when I say "managed", it mean the bean is not in the session context. The bean may correspond to an existing row in the database.

* If the instance you're merging is not managed, a new object is created and all properties are copied from your bean to the new instance.
* If there is a database object with the same id, update it, adding/removing rows from associated collections as well
* The instance copy is returned.
 
Please do read [this page](https://en.wikibooks.org/wiki/Java_Persistence/Persisting) for further info. Anyway, let's go on. Suppose you have, inside a ViewScoped bean like the following:

{% highlight java %}
    @Inject
    private EntityManager em;
    
    private Bean bean;
    
    public String addChild() {
        Child child = new Child();
        bean.addChild(child);

        save();
        return null;
    }

    public String save() {
        em.merge(bean);
        return null;
    }
{% endhighlight %}

And two buttons like:

{% highlight xml %}
    <h:commandButton action="#{bean.addChild}" value="Add Child" />

    <h:commandButton action="#{bean.save}" value="Save" />
{% endhighlight %}

When you click the "Add Child" button, a new "Child" database row is inserted. Okay!. However, when you click "Save" on a second request, the recently added child row is duplicated. Ouch! Why? Considering what I told you about the "merge" operation, let's think about it: You called "merge" on the "bean" instance. What happened to the instance? Absolutely nothing! The bean itself is kept intact.

* When you clicked "addChild", the "merge" operation was called. A new instance was created and all properties copied to it.
* The database operations were performed. More specifically, a database "UPDATE" was performed (considering the bean already existed in the database). 
* The "child" row was also inserted on the database and the database id was assigned to the child bean.
* Remember: All of this was done to the instance copy. 
* When you called "save", the merge was done to the original bean, which still doesn't have an id associated to the child instance, right? So, it will insert a new database row!

So, this duplication thing is easily solved with a single line of code:

{% highlight java %}
    public String addChild() {
        Child child = new Child();
        bean.addChild(child);

        save();
        return null;
    }

    public String save() {
        bean = em.merge(bean);
        return null;
    }
{% endhighlight %}

See how you reassigned "bean" to the result of the merge operation? This will guarantee that you always use the bean instance that really reflects all the changes you made to the database, thus avoiding the duplicate row.

## Careful with the "Hibernate" dependency versions!

You did everything nicely. You are always using the result of the "merge" operation. Nevertheless, you find out that a single call to a merge is duplicating rows! You try to find out WHAT. THE. FUCK. is going on. You apply all kinds of funky workarounds. You try to merge the children before merging the parent. It works for simple use cases just to find out hell breaks loose on more complex beans. You want to give up and just try to use something else. You want to obliterate JPA/Hibernate, crush JPA/Hibernate, destroy JPA/Hibernate. If you can, please do so! If you can't, take a closer look to the Hibernate versions you're using.

Let me tell you about my days of pain with this duplicate row issue. I was using the following dependencyManagement section to include the dependencies:

{% highlight xml %}
  <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.jboss.bom</groupId>
                <artifactId>jboss-javaee-6.0-with-hibernate</artifactId>
                <version>1.0.7.Final</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
{% endhighlight %}

Then, I was including the Hibernate dependency:

{% highlight xml %}
   <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>4.1.7.Final</artifactId>
   </dependency> 
{% endhighlight %}

This specific web project is run inside a JBoss EAP 6.2.0 server, which uses a different set of dependencies. Moreover, I was needlessly specifying the Hibernate version. I had a really hard time search for the "1.0.7.Final" version, and I thought this version was okay. That, however, was the problem! I miraculously found a decent match for my JBoss version. Then, really, all it took to fix this was to change the dependencyManagement section to:

{% highlight xml %}
  <dependencyManagement>
        <dependencies>
            <dependency>
               <groupId>org.jboss.bom.eap</groupId>
               <artifactId>jboss-javaee-6.0-with-hibernate</artifactId>
               <type>pom</type>
               <scope>import</scope>
               <version>6.2.0.GA</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
{% endhighlight %}

And remove the version from the Hibernate dependency:

{% highlight xml %}
   <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
   </dependency> 
{% endhighlight %}

And everything started working just fine, because now the Hibernate versions match the versions JBoss expects and uses. No damn duplicate rows! Bottom line: mind the dependency versions you're including. Things can get really really messy if you don't. Hope this helps the poor souls that face the dreaded pressure test, I mean, the dreaded duplicate row thing.
