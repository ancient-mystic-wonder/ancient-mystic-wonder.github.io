---
layout: post
title:  "Trimming JsonNode's null children (feat. recursion)"
date:   2018-03-26
categories: java 
---

Now this post may be a bit short, and to some, whimsical (I've always wanted to use that word). However I just feel the need to share this, to get this off my chest.

This week I encountered a fairly odd problem: I had to remove JSON fields that had all null children. For example, our Spring boot application was receiving this JSON value:

```
{
    "name": "Dale",
    "address": {
        "street": null,
        "country": null
    }
}
```


