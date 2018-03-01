---
layout: post
title:  "Adventures with Ad Hoc Joins in Spring (and AWS Redshift): Part 2"
date:   2018-02-27
categories: spring
---
_[Part 1]({{ site.baseurl }}{% post_url 2018-02-19-redshift-join-1 %}) explores the basics on using join fetch for Spring Data_<br>
_Part 2 explores more on using AWS Redshift entities with Spring_

In the last part, I explored the happy path of using join fetch for two or more entities in Spring Data. Now we shall see what happens when we add entities from my work's AWS Redshift into the mix.

So what's the deal with AWS Redshift? Well, at the time of this writing, my knowledge on AWS Redshift is limited, however what I do know is that it is possible for a Redshift table to not have any constraints. No primary key, foreign key, etc. which was exactly my situation at work.

This presents a problem for Spring's entities, as we shall see in a while.

<br>

## Attempting to join `Customer` and `Table`
I started with trying to join two entities, in this case we shall go with `Customer` and `Table`. Now since `Customer` no longer has a primary key, I decided to annotate ALL its fields with `@Id`, which was perfectly fine.

```java
@Entity
@Table(name = "customers")
@Data
public class Customer implements Serializable {

    @Id
    @Column(name = "id")
    private int id;

    @Id
    @Column(name = "name")
    private String name;

    @Id
    @Column(name = "age")
    private int age;

    ...

```
Problems will arise, however, once I started to annotate the `Table` entity with `@Ids`:

#### Attempt 1: Annotate all fields of the `Table` entity with `@Id`:

#### Attempt 2: Annotate just the join reference column(s) with `@Id`:

#### Attempt 3: Annotate everything EXCEPT the join reference column(s) with `@Id`:

So here we are, already stuck at joining two tables. What more when we decide to add the `Order` table to the party? We will be experiencing twice the problems above.

<br>

## Implementing ad hoc joins
After a lot of fiddling and googling, I decided that Spring Data was being to restrictive and that I had to give it up in order to get more control on my entities. Heck, the devil on my shoulder was even pushing me to just use `JdbcTemplate` (which I in fact did at work, just to prove to myself that it can actually be done). 

Thankfully I came across [this][article-ad-hoc] article on using `EntityManager.createQuery(...)` to join two entities. This way, I get to keep using JPA/Hibernate and my entities won't be wasted. However, since I will be defining the fields to select, I needed to create a separate `CustomerResponse` object to hold the values.

The `CustomerResponse` object is just a DTO that has all the combined fields of `Customer`, `Table`, and `Order`:
```java
public class CustomerResponse {
    private String name;
    private int age;
    private String tableNumber;
    private int tableCapacity;
    private String foodName;
}
```

However, the price was that I can no longer leverage Hibernate's `@OneToMany`, `@ManyToOne` collections on one entity (well, maybe you still can, but you would have to manually construct the object from the result set).

[article-ad-hoc]: https://www.thoughts-on-java.org/how-to-join-unrelated-entities/
[entities-link]: https://github.com/ancient-mystic-wonder/threejointest/tree/master/src/main/java/com/dtlim/threejointest/domain
