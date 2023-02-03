# URL-Shortening-Service-System-Design

<object data="http://yoursite.com/the.pdf" type="application/pdf" width="750px" height="750px">
    <embed src="http://yoursite.com/the.pdf" type="application/pdf">
        <p>Please download the PDF to view it: <a href="https://github.com/mohamedlotfe/URL-Shortening-Service-System-Design/blob/main/SD_Challenge_solution_Presentation.pdf">Download PDF</a>.</p>
    </embed>
</object>

1-  Case Study 
- An e-commerce startup has a very original idea, they want to create a URL shortening service. 
- A visitor enters a very long URL into a field and gets a shortened version of it. 
- Whenever the shortened version is used to request a web resource, the service will redirect to the original URL
￼
- https://medium.com/@sandeep4.verma
- Short URL : https://tinyurl.com/3sh2ps6v



2- System Requirements

- Functional Requirements:
    - Service should be able to create shortened url/links against a long url
    - Click to the short URL should redirect the user to the original long URL
    - Shortened link should be as small as possible
    - that Links can expire after a default timespan
    - Users can create custom url with maximum character limit (EX: 16)

- Non-Functional Requirements:
    - High availability:  Service should be up and running all the time 
    - scalable & efficient & Minimal latency: URL redirection should be fast and should not degrade at any point of time (Even during peak loads)
    - Service should expose REST API’s so that it can be integrated with third party applications
    - Shortened links should not be predictable

- Extended requirements
    - Service should collect metrics like most clicked links
    - Optional Once a shortened link is generated it should stay in system for lifetime
    - 

3- Traffic and System Capacity

- Traffic
    - Assuming 200:1 read/write ratio
    - Number of unique shortened links generated per month = 100 million
    - Number of unique shortened links generated per seconds = 100 million /(30 days * 24 hours * 3600 seconds ) ~ 40 URLs/second
    - With 200:1 read/write ratio, number of redirections = 40 URLs/s * 200 = 8000 URLs/s
    - 
- Storage
    - Assuming lifetime of service to be 100 years and with 100 million shortened links creation per month, 
    - total number of data points/objects in system will be = 100 million/month * 100 (years) * 12 (months) = 120 billion
    - Assuming size of each data object (Short url, long url, created_at and expiration_length_in_minutes) to be 500 bytes long, 
    - then total require storage = 120 billion * 500 bytes =60TB
    - 
- Memory
    - Following Pareto Principle, better known as the 80:20 rule for caching. (80% requests are for 20% data)
    - Since we get 8000 read/redirection requests per second, we will be getting 700 million requests per day:
    - 8000/s * 86400 seconds =~700 million
    - To cache 20% of these requests, we will need ~70GB of memory.
    - 0.2 * 700 million * 500 bytes = ~70GB
Type	Estimate
Writes (New URLs)	40/s
Reads (Redirection)	4K/s
Bandwidth (Incoming)	20 KB/s
Bandwidth (Outgoing)	2 MB/s
Storage (10 years)	6 TB
Memory (Caching)	~35 GB/day
Type	Estimate
Shortened URLs generated  (Writes New URLs) 	40/s
Total URL requests/Reads (Redirection)	8K/s
Storage(100 years)	60TB
Memory (Caching)	70GB
Bandwidth (Incoming)	20 KB/s
Bandwidth (Outgoing)	4 MB/s

4- High Level Design

- Following is high level design of our URL service.  
- We will optimise this further as we move along.
- Problems with above design :
* 		There is only one WebServer which is single point of failure (SPOF)
* 		System is not scalable
* 		There is only single database which might not be sufficient for 60 TB of storage and high load of 8000/s read requests
- To cater above limitations we :
* 		Added a load balancer in front of WebServers
* 		Sharded the database to handle huge object data
* 		Added cache system to reduce load on the database.
We will delve further into each component when we will go through the algorithms in later sections
- 
5-  URL Shortening Logic (Encoding)-Shortening Algorithm
With all these shortening algorithms, let’s revisit our design goals!
* Being able to store a lot of short links (120 billion)
* Our TinyURL should be as short as possible (7 characters)
* Application should be resilient to load spikes (For both url redirections and short link generation)
* Following a short link should be fast

To convert a long URL into a unique short URL we can use some hashing techniques like Base62 or MD5. We will discuss both approaches. 
- Technique 1 (base62 encoding)
    - A base is a number of digits or characters that can be used to represent a particular number.
    - base 62 are [0–9][a-z][A-Z] = total( 26 + 26 + 10 = 62)
    - Let’s do a back of the envelope calculation to find out how many characters shall we keep in our tiny url.
    - URL with length 5, will give 62⁵ = ~916 Million URLs
    - URL with length 6, will give 62⁶ = ~56 Billion URLs
    - URL with length 7, will give 62⁷ = ~3500 Billion URLs
    - So for 7 characters short URL, we can serve 62^7 ~= 3500 billion URLs which is quite enough in comparison to base10 (base10 only contains numbers 0-9 so you will get only 10M combinations). 
    - If we use base62 making the assumption that the service is generating 1000 tiny URLs/sec then it will take 110 years to exhaust this 3500 billion combination. 
    - We can generate a random number for the given long URL and convert it to base62 and use the hash as a short URL id. 
    - This is the simplest solution here, but it does not guarantee non-duplicate or collision-resistant keys.
- Technique 2 (MD5 Encoding)
    - MD5 also gives base62 output but the MD5 hash gives a lengthy output which is more than 7 characters. 
    - MD5 hash generates 128-bit long output so out of 128 bits we will take 43 bits to generate a tiny URL of 7 characters. 
    - MD5 can create a lot of collisions. For two or many different long URL inputs we may get the same unique id for a short URL and that could cause data corruption. 
    - So we need to perform some checks to ensure that this unique id doesn’t exist in the database already
- Technique 3 (Counter Approach)
    - In this approach, we will start with a single server which will maintain the count of the keys generated. 
    - Once our service receives a request, it can reach out to the counter which returns a unique number and increments the counter. 
    - When the next request comes the counter again returns the unique number and this goes on.
    - Counter(0−3.5 trillion)→base62encode→hash
    - The problem with this approach is that it can quickly become a single point for failure. 
    - And if we run multiple instances of the counter we can have collision as it's essentially a distributed system
    -  But again we will face a problem i.e. if one of the counters goes down then for another server it will be difficult to get the range of the failure counter and maintain it again. 
    - Also if one counter reaches its maximum limit then resetting the counter will be difficult because there is no single host available for coordination among all these multiple servers
    - To solve this issue we can use a distributed system manager such as Zookeeper which can provide distributed synchronization. 
    - Zookeeper can maintain multiple ranges for our servers.
    - Once a server reaches its maximum range Zookeeper will assign an unused counter range to the new server. This approach can guarantee non-duplicate and collision-resistant URLs. Also, we can run multiple instances of Zookeeper to remove the single point of failure.
- Technique 4 (Key Generation Service (KGS))
    - we can create a standalone Key Generation Service (KGS) that generates a unique key ahead of time and stores it in a separate database for later use. 
    - KGS will make sure all the keys inserted into key-DB are unique
    - How to handle concurrent access?
    - if there are multiple server instances reading data concurrently, two or more servers might try to use the same key
    - KGS can use two tables to store keys: one for keys that are not used yet, and one for all the used keys. 
    - As soon as KGS gives keys to one of the servers, it can move them to the used keys table. 
    - KGS can always keep some keys in memory to quickly provide them whenever a server needs them
    - Can each app server cache some keys from key-DB? Yes, this can surely speed things up. Although, in this case, if the application server dies before consuming all the keys, we will end up losing those keys. This can be acceptable since we have 68B unique six-letter keys.
    - Isn’t KGS a single point of failure? Yes, it is. To solve this, we can have a standby replica of KGS. Whenever the primary server dies, the standby server can take over to generate and provide keys.
    - 
7. Cache
- We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like Memcached, which can store full URLs with their respective hashes
- How much cache memory should we have? We can start with 20% of daily traffic and, based on clients’ usage patterns, we can adjust how many cache servers we need
- Since a modern-day server can have 256GB memory, we can easily fit all the cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot URLs.
- Which cache eviction policy would best fit our needs? When the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? Least Recently Used (LRU) can be a reasonable policy for our system
- 
8. Load Balancer (LB)
We can add a Load balancing layer at three places in our system:
* 		Between Clients and Application servers
* 		Between Application Servers and database servers
* 		Between Application Servers and Cache servers
9. Algorithm REST Endpoints
1-createURL(apiKey: string, originalURL: string, expiration?: Date): string
 Return OK (200), with the generated short_url in data
 api_key: A unique API key provided to each user, to protect from the spammers, access, and resource control for the user, etc.

2- getURL(apiKey: string, shortURL: string): string

Return a http redirect response(302)
Note : “HTTP 302 Redirect” status is sent back to the browser instead of “HTTP 301 Redirect”. A 301 redirect means that the page has permanently moved to a new location. A 302 redirect means that the move is only temporary. Thus, returning 302 redirect will ensure all requests for redirection reaches to our backend and we can perform analytics (Which is a functional requirement)
short_url: The short URL generated from the above function.
Return Value: The original long URL, or invalid URL error code.

3- deleteURL(apiKey: string, shortURL: string): boolean

10.Security
For security, we can introduce private URLs and authorization. A separate table can be used to store user ids that have permission to access a specific URL. If a user does not have proper permissions, we can return an HTTP 401 (Unauthorized) error.
We can also use an API Gateway as they can support capabilities like authorization, rate limiting, and load balancing out of the box.

11. Database
With database we have two options :
	a. Relational databases (RDBMs) like MySQL and Postgres
	b. “NoSQL”-style databases like MongoDB and Cassandra

  A) RDBMs
- We can use RDBMS which uses ACID properties but you will be facing the scalability issue with relational databases. 
- Now if you think you can use sharding and resolve the scalability issue in RDBMS then that will increase the complexity of the system. 
- There are 30M active users so there will be conversions and a lot of Short URL resolution and redirections. 
- Read and write will be heavy for these 30M users so scaling the RDBMS using shard will increase the complexity of the design when we want to have our system in a distributed manner. 
- You may have to use consistent hashing to balance the traffics and DB queries in the case of RDBMS and which is a complicated process. 
- So to handle this amount of huge traffic on our system relational databases are not fit and also it won’t be a good decision to scale the RDBMS.   B) Now let’s talk about NoSQL!
- The only problem with using the NoSQL database is its eventual consistency. 
- We write something and it takes some time to replicate to a different node but our system needs high availability and NoSQL fits this requirement. 
- NoSQL can easily handle the 30M of active users and it is easy to scale. 
- We just need to keep adding the nodes when we want to expand the storage. 
- 
9- Database
10- Final Design
11.Database cleanup
12 references 

12.Database cleanup
This is more of a maintenance step for our services and depends on whether we keep the expired entries or remove them. If we do decide to remove expired entries, we can approach this in two different ways:
Active cleanup
In active cleanup, we will run a separate cleanup service which will periodically remove expired links from our storage and cache. This will be a very lightweight service like a cron job.
Passive cleanup
For passive cleanup, we can remove the entry when a user tries to access an expired link. This can ensure a lazy cleanup of our database and cache.


￼
