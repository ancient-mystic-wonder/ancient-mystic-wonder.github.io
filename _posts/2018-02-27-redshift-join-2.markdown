---
layout: post
title:  "Adventures with Ad Hoc Joins in Spring (and AWS Redshift): Part 2"
date:   2018-02-27
categories: spring
---
_[Part 1]({{ site.baseurl }}{% post_url 2018-02-19-redshift-join-1 %}) explores the basics on using join fetch for Spring Data_<br>
_Part 2 explores more on using AWS Redshift entities with Spring_

In the last part, I explored the rather happy path of using join fetch for two or more entities in Spring Data. Now we shall see what happens when we add entities from my work's AWS Redshift into the mix.

So what's the deal with AWS Redshift? Well, at the time of this writing, my knowledge on AWS Redshift is limited, however what I do know is that it is possible for a Redshift table to not have any constraints. No primary key, foreign key, etc. which was exactly my situation at work.

This presents a problem for Spring's entities, as we shall see in a while.

<br>

## Attempting to join `Customer` and `RestaurantTable`
I started with trying to join two entities, in this case we shall go with `Customer` and `RestaurantTable`. Now since `Customer` no longer has a primary key, I decided to annotate ALL its fields with `@Id` - which required the use of either an embeddable ID or the `@IdClass` annotation. I chose to use the latter for my example:

```java
@Entity
@Table(name = "customers")
@Data
@IdClass(Customer.CustomerId.class)
public class Customer implements Serializable {

    static class CustomerId implements Serializable {
        private String name;
        private int age;
        private String tableNumber;
    }

    @Id
    @Column(name = "name")
    private String name;

    @Id
    @Column(name = "age")
    private int age;

    @Id
    @Column(name = "table_number")
    private String tableNumber;

    @ManyToOne
    @JoinColumn(name = "table_number", referencedColumnName = "table_number", insertable = false, updatable = false)
    private RestaurantTable table;

    // no orders yet, we'll deal with it later

}
```

So far, so good - running this will get us the Customers along with their corresponding RestaurantTable. Now, when I add the list of orders:

```java
--- in Customer.java ---
...

@OneToMany
@JoinColumn(name = "customer_name", referencedColumnName = "name", insertable = false, updatable = false)
private List<Order> orders;
```

this exception occurs during compiling:
```java
org.hibernate.AnnotationException: referencedColumnNames(name) of com.dtlim.threejointest.domain.Order.orders referencing com.dtlim.threejointest.domain.Customer not mapped to a single property
```

which happens because the `@OneToMany` annotation's `@JoinColumn` annotation is expecting the list of orders to be mapped to one (and only one) customer - therefore whatever's in `referencedColumnName` must be all the columns that compose the identity of `Customer` (in this case, all of them)

I attempted to fix this by using only the `Customer.name` field with as the ID, however I later realized that this solution will not work - customers with the same name are treated as one entity (and Hibernate apparently will use the same object/reference for entities with same ID).

For example, if my `Customer` and `RestaurantTable` tables' contents are the following:

#### Customer:

| Name | Age | Table Number |
|------|------|------|
| Paul | 40 | 1 |
| Paul | 10 | 2 |

#### RestaurantTable:

| Table Number | Capacity |
|------|------|
| 1 | 5 |
| 2 | 6 |

Our left join on `Customer` and `RestaurantTable` will leave us with:
```
[{"name": "Paul", "age": 40, "table_number": 1, "table": {"table_number": 1, "capacity": 5}}
{"name": "Paul", "age": 40, "table_number": 1, "table": {"table_number": 1, "capacity": 5}}]
```

instead of the expected
```
[{"name": "Paul", "age": 40, "table_number": 1, "table": {"table_number": 1, "capacity": 5}}
{"name": "Paul", "age": 10, "table_number": 2, "table": {"table_number": 2, "capacity": 6}}]
```

Note: after tinkering with my entity for a bit, I found out that setting both `Customer.age` and `Customer.tableName` as our IDs also compiles fine. (turns out, the `@OneToMany` association either expects the referenced columns to either be ALL the IDs or none of them - not sure if this is intended or a bug). However, same as above, Hibernate will treat customers that have both the same age and table number as one entity.

<br>

## Implementing ad hoc joins
After a lot of fiddling and googling, I decided that Spring Data was being to restrictive and that I had to give it up in order to get more control on my entities. Heck, the devil on my shoulder was even pushing me to just use `JdbcTemplate` (which I in fact did at work, just to prove to myself that it can actually be done). 

Thankfully I came across this [article][article-ad-hoc] on using `EntityManager.createQuery(...)` to join two entities. This way, I get to keep using JPA/Hibernate and my entities won't be wasted. However, since I will be defining the fields to select, I needed to create a separate `CustomerResponse` object to hold the values.

The `CustomerResponse` object is just a DTO that has all the combined fields of `Customer`, `RestaurantTable`, and `Order`:
```java
@Data
@AllArgsConstructor
public class CustomerResponse {
    private String name;
    private int age;
    private String tableNumber;
    private int tableCapacity;
    private String foodName;
}
```

However, the price was that I can no longer leverage Hibernate's `@OneToMany`, `@ManyToOne` collections on one entity (well, maybe you still can, but you would have to manually construct the object from the result set). It is up to the programmer on what he wants to do with the result set.

Thus, my Spring Data repository was replaced with this:

```java
@Repository
public class AdhocCustomerRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public List<CustomerResponse> findAllLeftJoined() {
        List<CustomerResponse> results = entityManager.createQuery(
                "select new com.dtlim.threejointest.dto.CustomerResponse(" +
                        "c.name, c.age, c.tableNumber, t.capacity, o.foodName) " +
                        "from Customer c left join RestaurantTable t on c.tableNumber = t.tableNumber " +
                        "left join Order o on c.name = o.customerName", CustomerResponse.class)
                .getResultList();

        return results;
    }

}
```
Not really the prettiest piece of work I have done, but it gets the job done. Note that I leveraged Hibernate's `select new` statement in order to construct the object in the query itself (of course, the `CustomerResponse` DTO has to have a constructor that takes those same parameters in order - thus the usage of the Lombok annoation `@AllArgsConstructor`).

The funny thing is, having the objects in the form of the DTO was also the form in which I needed the data at work (since we needed to show them in a table). If we were to return the original Entity (with `@ManyToOne` and `@OneToMany`) then I would have had to flatten them into multiple objects anyway. So I take this to be a blessing in disguise I guess.

## Conclusion
So now I have a working solution for our AWS Redshift entities at work - a "lower level" solution using `EntityManager` and ad hoc joins. The whole sample project can be found [here][repo]. Navigate to the `redshift` branch to see the work done on this post.

These two posts were the gist of the journey I went through to arrive at this (there were a heck of a lot more mistakes when I was solving it at work). Was it the best and/or optimal solution? Probably not, but for now it works. Feel free to contact me on any improvements and/or different solutions.

[article-ad-hoc]: https://www.thoughts-on-java.org/how-to-join-unrelated-entities/
[entities-link]: https://github.com/ancient-mystic-wonder/threejointest/tree/master/src/main/java/com/dtlim/threejointest/domain
[repo]: https://github.com/ancient-mystic-wonder/threejointest/tree/redshift
