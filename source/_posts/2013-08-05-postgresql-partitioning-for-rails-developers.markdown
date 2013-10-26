---
layout: post
title: "PostgreSQL partitioning for Rails developers"
date: 2013-08-05 10:29
comments: true
categories: [postgresql, ruby-on-rails]
---

To begin with, let me tell that partitioning is evil. If you can avoid it, please do. Buy more RAM if you can, to make your tables fit into memory. But if your tables become really large, partition them.

<!--more-->

How large is large? 

Your table is too large, when it can't fit in RAM entirely. You can find the size on disk of your largest tables using this query:

{% codeblock lang:sql %}
SELECT nspname || '.' || relname AS "relation",
    pg_size_pretty(pg_relation_size(C.oid)) AS "size"
  FROM pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
  ORDER BY pg_relation_size(C.oid) DESC
  LIMIT 10;
{% endcodeblock %}

My output is:

{% codeblock %}
                  relation                   |  size   
---------------------------------------------+---------
 public.tracks                               | 42 GB
 public.track_referers                       | 22 GB
 public.index_track_query_strings_on_url     | 20 GB
 public.track_query_strings                  | 18 GB
 audit.logged_actions                        | 14 GB
 public.audits                               | 10 GB
 public.batch_mail_activities                | 8276 MB
 public.tracks_pkey                          | 7609 MB
 public.index_track_query_strings_on_url_md5 | 6178 MB
 public.index_track_referers_on_url_md5      | 4452 MB
(10 rows)

{% endcodeblock %}

As you see, my largest table is 42 GB, and I have only 16 GB of RAM on my server, so the `tracks` table is a prime candidate for partitioning.

You are now convinced that you need to partition some of your tables, but you don't know where to start, and would like to understand this whole business. 

Let us begin from the beginning.

### What is partitioning?

Quoting from the [documentation](http://www.postgresql.org/docs/9.2/static/ddl-partitioning.html) 
{% blockquote %}
Partitioning refers to splitting what is logically one large table into smaller physical pieces
{% endblockquote %}

Yes, that simple. You just break your large table into smaller sub-tables. In this way you gain several advantages over having all your data in one monolithic über table:

 1.Your queries performance can be improved dramatically when the data you need is in one (or few) partitions.

 2.You can delete slices of your data by removing unneeded partitions, which is much faster than regular delete.

 3.You can archive your data by moving rarely used partitions to a cheaper storage, though it is an advanced usage of PostgreSQL.

### How it works in postgresql

#### How do I break the large table into smaller sub-tables, a.k.a partitions? 

Let's start from a regular way of creating tables, I suppose you already know how to do it. Let's create one!

{% codeblock lang:sql %}
create table billing_operations
(
  id serial primary key,
  service_id int not null, 
  user_id int not null,
  amount money not null,
  created_at timestamp not null
);
{% endcodeblock %}

Now, how do I partition this table? 

Postgres uses table inheritance to implement partitioning. It allows you to create a main, _master_ table and inherited tables. 




#### How do I insert data? 

#### How do I query it afterwards? 

#### How and why do I get the performance increase?

### Strategies (range, list, manual, automated)
### Index (query analyze)
### Routing (?)
### Performance (dates and numbers and shit)
### Rails migration (migration example both manual and automated)
### Gems(partitioned)
### Conclusion(why evil?)