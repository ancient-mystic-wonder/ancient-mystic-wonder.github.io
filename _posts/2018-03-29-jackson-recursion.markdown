---
layout: post
title:  "Trimming JsonNode's null children (feat. recursion)"
date:   2018-03-29
categories: java 
---

*If you just want to see the code, click [here](#code)*

Now this post may be a bit short, and to some, whimsical (I've always wanted to use that word). However I just feel the need to share this, to get this off my chest.

This week I encountered a rather odd problem: I had to remove (or nullify) JSON fields that had all null children. For example, our Spring boot application was receiving this JSON value:

```
{
    "name": "Dale",
    "address": {
        "street": null,
        "country": null
    }
}
```

which had to be converted to this:

```
{
    "name": "Dale",
    "address": null
}
```

after which I would just let ObjectMapper's `mapper.setSerializationInclusion(Include.NON_NULL)` do the job of removing the now turned null field, ultimately serializing the whole object as just:

```
{
    "name": "Dale"
}
```

<br/>

## Solution

Since I was converting the object as a Jackson `JsonNode`, I basically have a reference to the root of a tree whose nodes are JSON fields (you can read more about JsonNode [here][jsonnode-docs])

For example, for the object above, the root `JsonNode` has two children: the `JsonNode` "name":"Dale" and the `JsonNode` of "address", which further has two children: "city":null and "country":null.

Remembering our CS data structures, one of the ways to traverse and clean up this tree is by using the dreaded **recursion** (*dun* *dun* *duuuun*)

Which brings us back to why I'm writing this post in the first place: this is the first time in my 2+ years of development work that I actually used a recursive solution for real-life production code.

**YES THAT'S RIGHT RECURSION IS REAL BOYS AND GIRLS DON'T SKIP YOUR CS CLASSES** 

Joking aside, yes, I am aware that recursion is probably used behind the scenes on some/most libraries and Java components that we use. I just found it enlightening to be using it first-hand.

<br/>

## Code
Anyways, here's the whole code to nullify JsonNodes that have all null children (not yet tested for arrays):
```java
// returns true if all children are null
private static boolean removeNullChildren(JsonNode node) {

    if(node.isNull()) {
        return true;
    }

    boolean allChildrenAreNull = true;
    for (Iterator<String> iter = node.fieldNames(); iter.hasNext(); ) {
        String key = iter.next();
        JsonNode child = node.get(key);

        if(child.isObject()) {
            if(removeNullChildren(child)) {
                ((ObjectNode) node).set(key, null);
            } else {
                allChildrenAreNull = false;
            }
        } else if(!child.isNull()) {
            allChildrenAreNull = false;
        }
    }
    return allChildrenAreNull;
}
```

What the function basically does is this:  
1. iterates through a `JsonNode`'s children 
2. nulls out whoever is empty (has all null children), and 
3. returns true or false whether all of its children are null (including the ones nulled out from step 2)

[jsonnode-docs]: http://www.baeldung.com/jackson-json-node-tree-model
