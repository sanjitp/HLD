# Design Facebook's News Feed - Practice Notes

### Design Facebook's News Feed Guided Practice - February 21, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmluy642x00s007ad23exf5cy" -->

#### Key Takeaways

**Requirements**

**Core Entities**

* The three fundamental entities for a news feed system are User (who creates content), Follow (the relationship that determines whose posts appear in your feed), and Post (the actual content items). These form the minimum viable data model before adding features like likes or comments.

* When defining entities in system design interviews, start with a plain list of names at the appropriate level of detail. Avoid adding implementation specifics like field types or constraints too early, as this keeps the conversation focused on high-level architecture first.

**API**

* Authentication information like user IDs should come from session tokens or JWT headers rather than request body parameters. This prevents users from impersonating others by simply changing a userId field in the payload.

* RESTful endpoint naming should focus on resources rather than actions. Use POST /posts instead of POST /feed because you are creating a post resource. Use PUT /users/{userId}/followers instead of PUT /follow/{userId} because you are modifying the followers collection of a user resource.

* PUT is the appropriate HTTP verb for follow operations because following is idempotent. Calling follow multiple times produces the same result as calling it once, which matches PUT semantics better than POST.

**High Level Design**

* Always walk through the data flow verbally in system design interviews. Explain how a request travels from client through API gateway to services and database, even if your diagram is clear. This demonstrates you understand how components interact, not just what they are.

* For DynamoDB follow relationships, use userFollowing as partition key and userFollowed as sort key. Create a Global Secondary Index with reversed keys (userFollowed as PK, userFollowing as SK) to efficiently query both who someone follows and who follows them.

* Feed generation uses a pull (fan-out on read) approach with three steps: fetch list of followed users, retrieve their posts using a GSI on Posts table (creatorId as PK, createdAt as SK), then sort chronologically. This approach can face scaling challenges when users follow many accounts because posts are fetched at request time.

* When designing DynamoDB tables, identify your query patterns first. For Posts, you need to query by creator and time, so create a GSI with creatorId as partition key and createdAt as sort key to enable efficient chronological retrieval of a user's posts.

**Deep Dives**

* When a user creates a post in a precomputed feed system, use async worker pools with a message queue to fan out the post to all followers' feeds. This prevents the post creation request from blocking while millions of rows are written, keeping the write path fast for the user.

* For social media feeds with celebrities, use a hybrid approach where accounts with many followers (like 100k+ threshold) are flagged to skip precomputation. Regular users get precomputed feeds for fast reads, while celebrity posts are fetched in real-time and merged at read time to avoid writing to millions of feeds.

* Precomputed feeds solve the read latency problem by storing ready-to-display feeds in a table with userId as the partition key. Reads become simple lookups instead of expensive joins and sorting operations, but this trades off write amplification where every post creates multiple rows.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements for this system?

> The system should be highly available for viewing the feed , stale data permitted upto 1 min (eventual consistency) The system should support 2B users Post and viewing of feed should be fast ( < 500ms) User can follow any number of users

**Core Entities**

**Q:** What are the core entities of the system?

> User Follow Post

**API**

**Q:** What are the main REST APIs that this system will need?

> POST /posts-> Post 200 OK body: { content - attachments - }\
> PUT /users/{userId}/followers -> void 200 OK {}\
> GET /feed?pageSize={size}&cursor={timestamp} -> { items: Post[], cursor: string }

**High Level Design**

**Q:** How will users be able to create text posts?

> Users will make a request for creating a post, the request will go to the API gateway , where it will be authenticated/authorized and routed to the post service, which serves the request and stores the post info to the Database We will be using dynamo Db for this case for its simplicity and scalability

**Q:** How will users be able to friend/follow people?

> We can store the mapping in a follow table with userFollowing as the Partition key and userFollowed as the sort key Also we can create a GSI in the reverse manner. So this helps us with the querying to check if a user follows other user -> we check both the PK & SK (userFollowing: userFollowed) to get all the users a given user is following -> query with PK (userFollowing) , range query to get all the followers of a given user -> query with the PK of the GSI , range query

**Q:** How will users be able to view a feed of posts from people they follow? Don't worry about pagination yet.

> There are three things we need to do here to get a users feed\
> Fetching all the posts from the users will be tricky since we don't have any index on posts table So we can create a GSI on posts table createdId as PK and createdAt SK this will help use get all the posts form the users in a sorted manner. Fetch all the users followed by the given user Fetch all the posts posted by the followed users. Sort the posts chronologically.

**Q:** How will users be able to page through their feed? (i.e., infinite scroll - scroll down to see older posts without reloading the entire page)

> *No answer recorded*

**Deep Dives**

**Q:** How do we efficiently read feeds for users who follow thousands of accounts while maintaining low latency?

> In our current system , if a user is following thousands of users , generating the feed for that user will be led to high latency For a user following huge users , we can pre compute the feed for them, we can store it in a different table precomputed feed which will a list of post Ids for every user We can have the partition key as user Id Let's do some storage calculation Assuming each post will be of 10 bytes and to store 200 posts per user , 2kb and we have 2Billion users, that comes out to be 4tb , which any modern DB can handle This definitely improves the read performance but every time a post is created , we have to update multiple rows .

**Q:** How do we avoid write amplification when a celebrity user with millions of followers creates a post that needs to update millions of other users feeds?

> We will create async feed workers that are working off a shared queue (SQS) which writes to our precomputed feeds\
> We can select which user to precompute the feed for and for which we do not\
> For writes For a celebrity with 100million followers, we don't have to update to all the followers , rather we can have a flag in the follow table which tells us that this follow is not precomputed, so the async workers will ignore the requests for these users.\
> For reads When a requests for their feed, we can grab the partially precomputed feed from feed table and merge it with the recent posts from those accounts which aren't precomputed

​

​

​