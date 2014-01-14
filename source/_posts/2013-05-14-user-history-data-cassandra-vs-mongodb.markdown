---
layout: post
title: "User History Data: Cassandra vs. MongoDB"
date: 2011-07-10 22:47
comments: true
categories: [Scalability, NoSQL]
---
Hello, there. Today I want to write about a new project I am working on, and why I picked Cassandra over MongoDB. This post assumes you have a basic understanding of each technology. I also want to preface this post by saying that while I really like Cassandra for this use case, we have been happily and successfully using MongoDB since I transitioned most of our user data storage over from Postgres (and Hibernate hell), but this is a topic for another post.

This new project is about storing music 'listen' history across our web app and mobile clients. That is, whenever one of our users listens to a song, we want to track who played the song, what song was played, the context of where the song was played from (a playlist, a radio station, an album), and when the song was played. In Java, the object might look like this:

``` java
public class ListenHistoryEvent
{
	int userId;
	int songId;
	Enum contextType;
	String contextId;
	Date datePlayed;
}
```

Users need to be able to retrieve their own song history, and should be able to see the song history of their friends, but only 10-20 items at a time, in order. There will never be a case, where we need to pull multiple users' history from storage at once (range scans).

Considering we are a music service, it should not be surprising that our #1 API call is for streaming music, so whatever storage engine we choose will have to sustain a constant, high write rate. As for the reads, I expect a much lower rate since I would assume that users won't be checking their listen history after each listen (or perhaps even after every 10th or 100th listen). With that, I'm going into this project designing for a 1:10 read/write ratio.

Let's see how Cassandra and MongoDB compare for implementing this project and requirements.

<h3>CAP and Overall Design</h3>

I think a great place to start is with the CAP theorem. Cassandra is classified as an AP system, where MongoDB is classified as a CP system. For this project, I need a system that is always available to sustain a high volume of writes, and I would rather gain overall performance even it if means potentially sacrificing consistency on reads. Let's face it, 99% of users will likely be checking their listen history every 10-100 songs, if they do at all, but in the unlikely event that they are glued to their listen history page, and refreshing it maniacally as they listen to music, I would be OK if sometimes they didn't see the latest data (think Facebook news feed) as long as the storage tier is able to scale all of those writes.Cassandra and MongoDB are both 'P' systems because they scale by partitioning and distributing load across multiple physical servers. MongoDB is 'C' leaning, because assuming you do shard a DB/collection, there is always a specific and designated master node for each _id (document) being written. If that master (shard) goes down, and there is no replica to replace it, you've got nowhere to write that data.

Cassandra is slightly different, and hence it's 'A' leanings. While there are also designated nodes for each key (row) that is being written, none of these nodes need to actually be up for the cluster to accept writes for it. With hinted handoff, the cluster will always be available to accept writes.

This is what I am looking for.

<h3>Persistence</h3>

Not only does Cassandra offer better write availability, it also shines at write performance, which is critical for such a write-heavy project. As of MongoDB version 1.8, it still has that global write lock which would impact performance not only on the collection being written to, but other collections as well across the cluster. Cassandra's BigTable inspired data model which writes in an append-only fashion really shines here with much (much) lower contention.

<h3>Storage Format and Memory Management</h3>

Let's transition a bit and talk about schema for a moment. If I were to structure this in MongoDB, I'd have two obvious choices:

- Each listen event is a document
    - Pros
        - Simple to program to
	- Cons
    	- Paginated reading of listen events would require a compound index with userId and datePlayed.
        - That is a lot of overhead for data that is not often read.
- Each user is a document and has embedded listen events
	- Pros
   		- Organized - I prefer this structure as it keeps things together by user and leverages the benefits of the document format nicely.
    	- Simple $slice operator can be used to handle pagination
	- Cons
    	- 16MB doc limit could create an issue for users who listen to a lot of music. Assuming 100 bytes/embedded listen event, that's about 160K songs/user.
        - Managing this array will create some code complexity until the 10Gen folks implement pushing to fixed sized arrays. I also wouldn't want to transfer 16MB of document over the wire to the app server to implement this logic.

So, while it's definitely possible to implement this with Mongo in several ways each with their pros and cons, I think Cassandra's model is a better fit for this project. I've chosen what some call the 'wide row' model where each user is a row, and each song listen event is a column in a single Column Family with the column key as the listening time and the column value is the event payload. It would look as follows in JSON format:

``` javascript
SongHistory =
{ // ColumnFamily
userId1:{ // Row in the SongHistory CF, using RandomPartitioner
    TimeUUID1: [Song History Data], //Each column has TimeUUID as key, and are sorted by key
    TimeUUID2: [Song History Data] //Each column value is a song listen  },
userId2: {// Another row
    TimeUUID1: [Song History Data]  }
,}
```

Rather than having MongoDB's 16MB document limit, Cassandra has a liberal 2GB column limit per row, which gives me quite a bit more headroom. Using Cassandra's Random Partitioner will also give me great dispersion on my writes across my users (using MD5 on the row key, which is the userId). The column keys will be sorted using the TimeUUIDType comparator, which will keep my columns in order. Getting this level of dispersion using MongoDB sharding would likely require a separate special field on the document for this.

The big win here is that with this schema, Cassandra need not deserialize the entire column to read or write song events which will save a lot of server resources across millions of users.

Another point to quickly mention on this topic is how MongoDB and Cassandra each handle caching, another area where I think Cassandra shines for this project.

Because MongoDB uses mmap files, it will be caching all of our active listening users song history in memory. Since most of our users probably won't be reading this data often, if at all, it would likely be a wasted resource, and if there is more important data on the same MongoDB cluster, but which gets written to less often, it would get pushed out from all of these writes. This is speculation and would have to be measured. From what I've read, Cassandra has a more granular caching structure which would cache individual rows and individual columns on an LRU basis. Given that users will first be reading their most recent writes, I think this is a good model for this project. As I load test this and do read simulations, I'll be very interested in learning more how to tune my caching strategy on Cassandra.

<h3>Conclusion</h3>
Well, I hope this post was helpful to you. As I get further along in the project, I'll post some updates. If you have any questions, I'd love to hear from you.
