---
layout: post
title:  "Adventures with Ad Hoc Joins in Spring (and AWS Redshift): Part 1"
date:   2018-02-19
categories: spring
---
_Part 1 explores the basics on using join fetch for Spring Data_<br>
_Part 2 explores more on using AWS Redshift entities with Spring_

These past two weeks I have been dealing with a rather unique problem at work: joining three entities from an AWS Redshift database using Spring Data.  Now, the "old and two weeks less wise" version of me proclaimed that this would be a breeze - I mean all I have to do is annotate some `@ManyToOne`s and `@OneToMany`s and let Spring work its magic right? 

Well it turns out this hole was a lot deeper than I thought, and the only way out is learning more about Spring and JPA/Hibernate the hard way. 

This writeup is all about the mistakes, hurdles, intricacies, and "AHA!"s that I have encountered while solving this problem. The entities used here are not the same as the ones I used at work (for confidentiality), however what I will be doing with them very closely mirrors what we had to do at work. 

You can clone the repository [here][git-repo].

## The Problem
My project uses a classic example of SQL entities for joins: a restaurant. I have created three entities: Customer, Table, and Order. A UML diagram of these is provided below. The relationship between them is described as follows:
- multiple Customers sit in one Table
- a Customer can have multiple Orders

![Restaurant UML Diagram]({{ "/assets/redshiftjoin/uml.png" | absolute_url }})

Note that I intentionally omitted all foreign keys from my tables in order to simulate the ad hoc, unrelated nature of the tables I had to join at work. You can take a closer look at the code for the entities [here][entities-link]

As for what the output of my program is supposed to be: to keep things simple, the objective is to list a left outer join between the Customer, Table and Order tables.

## Doing three joins
Let's start by forgetting about AWS Redshift for a moment and tackling the basics - how are we going to do a three-way join with Spring Data? Turns out, making it efficient is not as straightforward as I thought. At first, I decided to code my `Customer` class as follows:

```java
@Entity
@Table(name = "customers")
@Data
public class Customer implements Serializable {

    @Id
    @Column(name = "id")
    private int id;

    @Column(name = "name")
    private String name;

    @Column(name = "age")
    private int age;

    @Column(name = "table_number")
    private String tableNumber;

    @ManyToOne
    @JoinColumn(name = "table_number", referencedColumnName = "table_number", insertable = false, updatable = false)
    private RestaurantTable table;

    @OneToMany
    @JoinColumn(name = "customer_name", referencedColumnName = "name", insertable = false, updatable = false)
    private List<Order> orders;
}
```
which is completely fine - many customers are mapped to one table, one customer is mapped to many orders.
However, doing a simple findAll() from `CustomerRepository` will yield the following SQL statements:

```
Hibernate: /* select generatedAlias0 from Customer as generatedAlias0 */ select customer0_.id as id1_0_, customer0_.age as age2_0_, customer0_.name as name3_0_, customer0_.table_number as table_nu4_0_ from customers customer0_
Hibernate: /* load com.dtlim.threejointest.domain.RestaurantTable */ select restaurant0_.id as id1_2_0_, restaurant0_.capacity as capacity2_2_0_, restaurant0_.table_number as table_nu3_2_0_ from restaurant_tables restaurant0_ where restaurant0_.table_number=?
Hibernate: /* load com.dtlim.threejointest.domain.RestaurantTable */ select restaurant0_.id as id1_2_0_, restaurant0_.capacity as capacity2_2_0_, restaurant0_.table_number as table_nu3_2_0_ from restaurant_tables restaurant0_ where restaurant0_.table_number=?
Hibernate: /* load com.dtlim.threejointest.domain.RestaurantTable */ select restaurant0_.id as id1_2_0_, restaurant0_.capacity as capacity2_2_0_, restaurant0_.table_number as table_nu3_2_0_ from restaurant_tables restaurant0_ where restaurant0_.table_number=?
Hibernate: /* load com.dtlim.threejointest.domain.RestaurantTable */ select restaurant0_.id as id1_2_0_, restaurant0_.capacity as capacity2_2_0_, restaurant0_.table_number as table_nu3_2_0_ from restaurant_tables restaurant0_ where restaurant0_.table_number=?
Hibernate: /* load com.dtlim.threejointest.domain.RestaurantTable */ select restaurant0_.id as id1_2_0_, restaurant0_.capacity as capacity2_2_0_, restaurant0_.table_number as table_nu3_2_0_ from restaurant_tables restaurant0_ where restaurant0_.table_number=?
Hibernate: select orders0_.customer_name as customer2_1_0_, orders0_.id as id1_1_0_, orders0_.id as id1_1_1_, orders0_.customer_name as customer2_1_1_, orders0_.food_name as food_nam3_1_1_ from orders orders0_ where orders0_.customer_name=?
Hibernate: select orders0_.customer_name as customer2_1_0_, orders0_.id as id1_1_0_, orders0_.id as id1_1_1_, orders0_.customer_name as customer2_1_1_, orders0_.food_name as food_nam3_1_1_ from orders orders0_ where orders0_.customer_name=?
Hibernate: select orders0_.customer_name as customer2_1_0_, orders0_.id as id1_1_0_, orders0_.id as id1_1_1_, orders0_.customer_name as customer2_1_1_, orders0_.food_name as food_nam3_1_1_ from orders orders0_ where orders0_.customer_name=?
Hibernate: select orders0_.customer_name as customer2_1_0_, orders0_.id as id1_1_0_, orders0_.id as id1_1_1_, orders0_.customer_name as customer2_1_1_, orders0_.food_name as food_nam3_1_1_ from orders orders0_ where orders0_.customer_name=?
Hibernate: select orders0_.customer_name as customer2_1_0_, orders0_.id as id1_1_0_, orders0_.id as id1_1_1_, orders0_.customer_name as customer2_1_1_, orders0_.food_name as food_nam3_1_1_ from orders orders0_ where orders0_.customer_name=?
```
Which I later found out was the dreaded "n+1 select queries" problem - caused by Hibernate individually querying every foreign object related to that entity (in this case - `Table` and `Order`) using whatever you put in the `@JoinColumn` as the where clause for the search queries.

_i.e. since I used `@JoinColumn(name = "table_number", referencedColumnName = "table_number"...)` Hibernated queried using `where restaurant0_.table_number=?`_

<br>

After a bit of googling, I figured that the solution for this was using a "join fetch", which can be implemented in two ways: 

#### 1. `@Query` with "join fetch"
The first one is to use the `@Query` annotation with the Spring Data repository and doing a "join fetch" between entities:
```java
@Query("SELECT c FROM Customer c LEFT JOIN FETCH c.table LEFT JOIN FETCH c.orders")
List<Customer> findAllUsingJoinFetch();
```
which yields the following query:
```
Hibernate: /* SELECT c FROM Customer c LEFT JOIN FETCH c.table LEFT JOIN FETCH c.orders */ 
select customer0_.id as id1_0_0_, restaurant1_.id as id1_2_1_, orders2_.id as id1_1_2_, customer0_.age as age2_0_0_, customer0_.name as name3_0_0_, customer0_.table_number as table_nu4_0_0_, restaurant1_.capacity as capacity2_2_1_, restaurant1_.table_number as table_nu3_2_1_, orders2_.customer_name as customer2_1_2_, orders2_.food_name as food_nam3_1_2_, orders2_.customer_name as customer2_1_0__, orders2_.id as id1_1_0__ 
from customers customer0_ 
left outer join restaurant_tables restaurant1_ on customer0_.table_number=restaurant1_.table_number 
left outer join orders orders2_ on customer0_.name=orders2_.customer_name
```

<br>

#### 2. Using `javax.persistence.criteria.Root.fetch()` from the Criteria API`
When using the Criteria API, one can call the `fetch()` on the entities that you wish to do the fetch join with:
_note: the `Customer` object has two properties `table` and `orders`_
```java
public static Specification<Customer> leftJoinedAll() {
    return (root, query, cb) -> {
        root.fetch("table", JoinType.LEFT);
        root.fetch("orders", JoinType.LEFT);
        return cb.conjunction();
    };
}
```
which yields the following query:
```
Hibernate: /* select generatedAlias0 from Customer as generatedAlias0 left join fetch generatedAlias0.table as generatedAlias1 left join fetch generatedAlias0.orders as generatedAlias2 where 1=1 */ 
select customer0_.id as id1_0_0_, restaurant1_.id as id1_2_1_, orders2_.id as id1_1_2_, customer0_.age as age2_0_0_, customer0_.name as name3_0_0_, customer0_.table_number as table_nu4_0_0_, restaurant1_.capacity as capacity2_2_1_, restaurant1_.table_number as table_nu3_2_1_, orders2_.customer_name as customer2_1_2_, orders2_.food_name as food_nam3_1_2_, orders2_.customer_name as customer2_1_0__, orders2_.id as id1_1_0__ 
from customers customer0_ 
left outer join restaurant_tables restaurant1_ on customer0_.table_number=restaurant1_.table_number 
left outer join orders orders2_ on customer0_.name=orders2_.customer_name 
where 1=1
```

As we can see, both methods successfully obtained the objects in just one query, which is what we wanted. (the Criteria API method has an extra `where 1=1` from using `cb.conjunction()` since I do not know how to return a `Predicate` object without any where clause)

## Conclusion

In this post I have shown the results, logs and some code on how I discovered a method for joining tables efficiently using join fetch. However this is only possible and quite trivial because of the entities having a primary key.

For the next part, we will dive into what exactly happens when I take away our precious primary key constraints and how it will affect our joins. You can view the post here

[git-repo]: https://github.com/ancient-mystic-wonder/threejointest
[entities-link]: https://github.com/ancient-mystic-wonder/threejointest/tree/master/src/main/java/com/dtlim/threejointest/domain
