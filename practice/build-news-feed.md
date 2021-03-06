# Source
* https://stackoverflow.com/questions/4162020/how-can-i-improve-this-php-mysql-news-feed
* https://www.quora.com/What-are-the-best-practices-for-building-something-like-a-News-Feed

# Areas
Main scenarios are below,

1. User can post status.
2. User can see news feed (from friends).

## Analysis
1. There're a few entities mentioned in scenarios: `users` and `feeds`.
2. A few relationships,
    * a user can post a feed
    * a user can add other users as friends
    * user can see feeds from other users.
3. Services
    * User profile management
    * User friends service
    * User post feed service
    * news feed service
 
 Base on #1 and #2, we can define the underlying data schema.
 
 * User table: store user profile information. UserId, name, gender.
 * User friends table: store user friend relationship table. UserId, FriendUserId.
 * Feed table: UserId, FeedId, FeedContent.
 
 Going back to #3, we would know which data used by each service,
 
 * User profile uses `User` table.
 * User friends service uses `User friends` table.
 * User post feed service uses `Feed` table.
 * News feed service service would use both `user friends` and `feed` table.
 
 ### Concerns
 1. The 4 services don't call other services yet. It uses the underlying DB directly. Is it right? 
    * one join to get all data including feed content?
    * one join to get candidate feed id list. Use another service to return feed content itself. This solution is better because
      * the backend can get a longer list of feed candidate and the front end can decide how many to show. When need to show, just get the content from cache based on the feed id.
      * the relationship table could have just two columns `userid` and `feedid`. The feed content can save in other DBs. In other words, we use this table to store feed metadata and another table or source to store the content itself.

## Deep Dive
1. Number: 1B users, each user has 100 friends, each user posts a feed every day in average. 
      * `user friends` has 100B rows? E.g., A -> B, B -> A. If we store only one for the friendship (e.g., A->B), would it add complexity when query a user's friends?
      * `feed table`: 1B rows every day to grow. too big.
      * To see feeds, join `user friends` table with `feeds` table? too big and slow.
2. First of all, we need to shard the data. It is just too big. How to shard?
      * randomly shard based on hash id. It seems alright. What's the cons?
      * partition by city because friends tend to live in the same city. **Question**: how to handle popular cities which have much more people than small cities?
      * Partition by user name? Same issue as city. How to avoid `hot` names? Auto sharding (like auto load-balancing)?
3. Bottleneck: reading news feed is too slow because of the big join. Write seems alright.
      * How to get faster? Join was used because related data are separated. If we can make them together, the query would be very fast. Basically it means we need to remove join, e.g., denormalization. It is called fanout. A user can have a list of feeds from all users.
         * content update - it is fine because we use feed id as reference.
         * feed is deleted - How can we make sure the data is consistent? We need to update the list from all friends.
         * a list of feeds: it can be represented as multiple rows in the table, or a key-value pair in redis.
      * write is ok then? When a user has a lot of friends, the write could be very slow. That's why Twitter uses a hybrid system. VIP user and normal user. VIP user will still uses join operation given that their feeds are relatively small.
4. How do we do ranking? The easiest way is to use timestamp (which has to be part of metadata table). Because timestamp never changes, new feed is just appened to the list. Cost is low. What if a new feed affects others' ranking? Updating all related feed ranking could be slow. Btw, we need a ranking service here.
5. Where do we use cache?
      * `Feed content service` and `user profile service` make sense. Redis is a perfect user case here. We need to use write-through cache because the write cost is usually low.
      * `user friends service` can use cache too. 
      * `feeds list service` - can we use cache? whenever a new feed is added, we can async update the new ranking and also the cache. If cache doesn't exist, we can recalculate the ranking.
6. Redundant
      * write - always write to two masters, or master-active, or masterless servers? How do we decide?
7. CAP - a user doesn't need to see the result immediately so availability is more important than consistency in this case. Eventual consistency (i.e., see the result finally) is good enough.
      * note, the user must see his status update immediately so it must be sync. Fan-out could be async though.

## Data Schema
* `user table`: user_id, name
* `user friendly table`: user_id, friendly_user_id
* `feed metadata`: user_id, feed_id, datetime (use server time)
* `feed content`: feed_id, feed_content
* `feed fanout table`: user_id, feed_id

### Details
1. primary id could be int or UUID. https://stackoverflow.com/questions/30461895/the-differences-between-int-and-uuid-in-mysql
      * int is unique in that specific DB table. Not unique universally. (4 bytes) bitInt would be 8 bytes. https://stackoverflow.com/questions/5634104/what-is-the-size-of-column-of-int11-in-mysql-in-bytes
      * UUID is unique universally. Performance is ok. Unnoticable slow. More space (16 bytes). Note, uuid is usually 36 chars when represent as string, but to store, it is 16 bytes in UUID version1.
      
## Something I missed here
1. Uses a background service to generate news feed for all users every 5 minutes. The result can be used as cache. Why not on-demand?
2. We return all feeds to user. It should paginate, i.e., the web api should accept an offset for that.

## Learning
### Twitter
* One tweet one row.
* How to partition? 
   * partition by primary id (int): P1 has 20 and 22 and P2 has 21 and 23. When query recent tweets, it has to go through N partitions.
   * partition by user id: when query tweet by primary id, it has to go through N partitions.
   * current solution: partition by date. Search latest partition until collect enough data.
* Low latency -> partition and index.
* Locality - new tweets are requestd most frequently so usually only 1 partition is checked.
### Timeline
* Naive SQL join: extremely slow if there're a lot of friends or index cannot be in RAM.
* Current implementation: the timeline is stored in memcached; fanout offline (low latency SLA); timeline has length limit; if cache miss, use join.
* Memory Hiearchy
   * Fanout to disk: too many IOs. Rebuild from other data store is not too expensive.
   * Fanout to memory: much faster.
### User Friends Table
* Partitioned by user id
* Edge stored in both `forward` and `backward` directions. E.g., it stores both A->B and B->A.
* Write operations are idempotent: retry until success.
* Partition, replicate and index: most used.
### Search
* Original design: maintain a table term_id + doc_id. It is like inverted indexing document. It says the index may not fit in RAM (weird)
* Current implementation: partition by time. old searched term is in old partitions.
* Future: partiton and merge. Something like map reduce.
### Others
* All data for real-time queries MUST be in memory; disk is for writes only.
* Some problems can be solved with pre-computation; but a lot cannot.
### 
