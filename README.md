# What is Web Crawler?
- A web crawler is a software program which browses the World Wide Web in a methodical and automated manner. 
- It collects documents by recursively fetching links from a set of starting pages. 
# Usage of Web crawlers
- To create index on downloaded page by Search engines to perform faster searches
- To test web pages and links for valid syntax and structure.
- To monitor sites to see when their structure or contents change.
- To maintain mirror sites for popular Web sites.
- To search for copyright infringements.
- To build a special-purpose index, e.g., one that has some understanding of the content stored in multimedia files on the Web.
# Requirements and Goals of the System
## Functional requirement
- Let’s assume we need to crawl all the web.
## Non functional requirement
- Scalability
    - Our service needs to be scalable such that it can crawl the entire Web and can be used to fetch hundreds of millions of Web documents.
- Extensibility
    - Our service should be designed in a modular way with the expectation that new functionality will be added to it. 
    - There could be newer document types that need to be downloaded and processed in the future.
- Respect robots.txt to exclude urls
## Volume Constraints 
- Number of pages to crawl in 4 weeks: 15 billion
# Single machine Design
On a single machine, this problem is deduced to a simple graph search problem, which can be solved with a simple BFS algorithm :
```
queue = { ... }  # initial url set
seen = {}
for url in queue : 
   break if goes too deep
   page = download(url)
   urls = extract_url(page)
   for url in urls :
     if url not in seen : 
        seen.add(url)
        queue.add(url)
```
# Distributed System design 
![](https://docs.google.com/drawings/d/e/2PACX-1vTclxHZRMT0zQMuQAD_FN7trcs2thsmZD6ar2weYEpgJKuDTLHY1biLQa6tDHFrdYJ1ioWKutnP1jL-/pub?w=960&h=600)
## Seed Urls
This is distributed messaging queue to hold all the urls needs to be processed. 
### Schema
```
{
    url: string 120 chars ~ 240 bytes
}
```
### Storage estimate
- Size of one message: 240 bytes
- Size of 15 billions messages: 15 billion * 240 bytes ~ 3.6TB
- Limits per server: 1TB
- Number of shards: 4
- shard key: url 
## HTML Fetcher
- The purpose of a fetcher module is to download the document corresponding to a given URL using the appropriate network protocol like HTTP.
- Webmasters create robot.txt to make certain parts of their websites off-limits for the crawler. 
- To avoid downloading robot.txt file on every request,  this module can maintain a fixed-sized cache mapping host-names to their robot’s exclusion rules.
- After download, it will write document on object storage
### App Scale
- Number of urls in 4 weeks: 15 billion
- Number of urls in a second: 6200 per sec
- Concurrent thread process by a server: 100
- Number of messages can be processed concurrently: 100
- Number of App server: 6200/100 ~ 62
## Document Stream
- This is distributed messaging queue to hold all the message for downloaded documents. 
### Message Schema
Following is schema for message
```
{
    documentId: sha256 128 bytes
    filePath: string 240 bytes
}
```
### Database Storage Estimate
- Size of message: 368 bytes + 2 * 20 bytes ~ 388 bytes ~ 400 bytes
- Number of urls in 4 weeks: 15 billion
- Storage for messaging queue: 15 billion * 400 bytes ~ 6Tb
- Storage per server: 1Tb
- Number of shards: 6
- shard key: documentId
### Object Storage Estimate
- Size of each document: 100kb
- Number of urls in 4 weeks: 15 billion
- Total storage : 15 billion * 100kb = 1.500  pb
- Storage per server : 1Tb
- Number of shard: 1500
- shard key: documentId
## Document Dedupe
- Many documents on the Web are available under multiple, different URLs.
- There are also many cases in which documents are mirrored on various servers.
- Both of these effects will cause any Web crawler to download the same document multiple times.
- To prevent the processing of a document more than once, we perform a dedupe test on each document to remove duplication.
- To perform this test, we can calculate a 64-bit checksum of every processed document and store it in a database. 
- For every new document, we can compare its checksum to all the previously calculated checksums to see the document has been seen before.
- We can use MD5 or SHA to calculate these checksums.
### Message Schema
Following is schema for message
```
{
    documentId: sha256 128 bytes
    filePath: string 240 bytes
}
```
### Storage Schema
Following is schema for storage
```
{
    documentId: sha256 128 bytes
    checksum: sha256 128 bytes
}
```
### Database Storage Estimate
- Size of message: 256 bytes + 2 * 20 bytes ~ 196 bytes ~ 300 bytes
- Number of urls in 4 weeks: 15 billion
- Storage for messaging queue: 15 billion * 300 bytes ~ 4.5Tb
- Storage per server: 1Tb
- Number of shards: 5
- shard key: documentId
### App Scale
- Number of urls in 4 weeks: 15 billion
- Number of urls in a second: 6200 per sec
- Concurrent thread process by a server: 100
- Number of messages can be processed concurrently: 100
- Number of App server: 6200/100 ~ 62
## URL Extractor 
- This module will extract all links from the page. 
- Each link is converted into an absolute URL and tested against a user-supplied URL filter to determine if it should be downloaded. 
### App Scale
- Number of urls in 4 weeks: 15 billion
- Number of urls in a second: 6200 per sec
- Concurrent thread process by a server: 100
- Number of messages can be processed concurrently: 100
- Number of App server: 6200/100 ~ 62

## URL Filter 
- This is distributed messaging queue to hold all the extracted urls. 
### Message Schema
Following is schema for message
```
{
    url: string 240 bytes
}
```
### Message Storage Estimate
- Size of message: 140 bytes + 1 * 20 bytes ~ 160 bytes
- Number of urls in 4 weeks: 15 billion
- Storage for messaging queue: 15 billion * 160 bytes ~ 2.4Tb
- Storage per server: 1Tb
- Number of shards: 3
- shard key: url
## URL Dedupe
- While extracting links, any Web crawler will encounter multiple links to the same document. 
- To avoid downloading and processing a document multiple times, a URL dedupe test must be performed on each extracted link before adding it to the URL seed queue.
- To perform the URL dedupe test, we can store all the URLs seen by our crawler in canonical form in a database. 
- To save space, we do not store the textual representation of each URL in the URL set, but rather a fixed-sized checksum.
- To reduce the number of operations on the database store, we can keep an in-memory cache of popular URLs on each host shared by all threads. 
- The reason to have this cache is that links to some URLs are quite common, so caching the popular ones in memory will lead to a high in-memory hit rate.
### Can we use bloom filters for deduping?
- Bloom filters are a probabilistic data structure for set membership testing that may yield false positives. 
- A large bit vector represents the set. 
- An element is added to the set by computing ‘n’ hash functions of the element and setting the corresponding bits. 
- An element is deemed to be in the set if the bits at all ‘n’ of the element’s hash locations are set. 
- Hence, a document may incorrectly be deemed to be in the set, but false negatives are not possible.
- The disadvantage of using a bloom filter for the URL seen test is that each false positive will cause the URL not to be added to the frontier and, therefore, the document will never be downloaded. 
- The chance of a false positive can be reduced by making the bit vector larger.
### Database Storage Estimate
- Size of message: 128 bytes + 1 * 20 bytes ~ 148 bytes ~ 150 bytes
- Number of urls in 4 weeks: 15 billion
- Storage for messaging queue: 15 billion * 150 bytes ~ 2.25Tb
- Storage per server: 1Tb
- Number of shards: 3
- shard key: url
### App Scale
- Number of urls in 4 weeks: 15 billion
- Number of urls in a second: 6200 per sec
- Concurrent thread process by a server: 100
- Number of messages can be processed concurrently: 100
- Number of App server: 6200/100 ~ 62