# SYS-402 Assessment 2 - Distributed `hearty-store`

The goal of this assessment is to evaluate your understanding on fundamental designs and implementations of distributed file systems or not.

This assessment will let you reconstruct the `hearty-store` and make it distributed. For simplicity, we would like to make it a multiple-client/single-server system first. In the upcoming assessment, you will enhance it into a multiple-client/multiple-server system.

For this assessment, you do not need to support all the `hearty-store` functionalities.

## Task #1 - The Server and Clients for `hearty-store` (40 points)
The server (`hearty-store-server`) will be the long-running process listening to the client request. 

Instead of initializing the resource through `hearty-store-init`, you should be able to initialize the resource with the initialization part of the server code. Whenever your `hearty-store-server` is run, you can now proceed to start to listen to the clients' requests.

The server must be able to handle the following **calls** from clients:
1) `hearty-store-init`: is a command that allows a client to create a store instance on the server machine. (Note: You will need to edit this in Assessment 3 so that you are allowed to create an instance on other server machines.)
2) `hearty-store-put`: is a command that allows a client to store a file in an instance.
3) `hearty-store-get`: is a command that allows a client to retrieve a file from an instance.
4) `hearty-store-list`: is a command that allows a client to list all the available stores.
5) `hearty-store-destroy`: is a command that allows a client to destroy an available store.

The **calls** must be implemented with either:
- [rpcgen](https://docs.oracle.com/cd/E19683-01/816-1435/rpcgenpguide-21470/index.html) (for C) - You need to make it a concurrent server by using `-A`
- [gRPC](https://grpc.io/) (for C++, Rust; recommeded but need to learn)

After you have done with the server, you also need to implement all the client programs corresponding to the **calls**.

### Naively Supporting Concurrent Accesses
In reality, you may need to use locks/latches. However, for simplicity (even though not efficient), the server can only handle one client at a time. More specifically, when each client request arrives at the server code, the request will need to try acquire the lock immediately. When there is another incoming request, the incoming request will get rejected instantly (given the lock has been acquired). This means whenever the server is done serving the request, the lock must be released to serve the upcoming requests.

You will need to create the *global lock*. I recommend you to use `std::atomic<bool>` or `std::mutex` to do so, since you will be more familiar with this than the other locking primitives. However, if you use C, you may want to use `mutex`. At the conceptual level, you need to declare a global mutex variable in the server code.

```
std::mutex global_lock;
```

Then, whenever you are starting to service the request and ending to service it, you need to acquire and release the lock respectively.

```
global_lock.lock();

// Service the request

global_lock.unlock();
```

However, if we use `lock()`, the request will be waiting until the `global_lock` gets unlocked. We instead want to reject the request. Therefore, we will use another function call `try_lock()` and if we are able to lock, we will proceed. If not, we will retreat.

```
if (global_lock.try_lock()) {

    // Service the request

    global_lock.unlock();
}
```

## Task #2 - Write-Back Caching on Clients (30 points)
Each client must be able to cache files especially when the files are needed to be access multiple times consecutively.

### Step #1 - Simple (Write-Through) Caching
You should start by creating a simple write-through cache first.

A single client will be able to cache at most 8 files/blocks on its local disk. Whenever the client wants to access the file (through `hearty-store-get` and `hearty-store-put`), the client will check the cache first. If the cache contains the file, the client does not need to send a remote request.

However, if the file is not in the cache, the client needs to send a request to the server. Once the file is arrived, the client needs to cache it in the caching area. **One of the issues is that the caching area might be full.** If it is full, the client needs to remove replace an old file with the newly-arrived one. The strategy to replace is called *Cache Replacement Policy*. You will need to implement the policy, namely *First-In-First-Out (FIFO)*, which is one of the simplest policies and not quite efficient. If you want to make it more efficient, you may want to use *Least Recently Used (LRU)*, *CLOCK*, *2Q*, and [much more](https://en.wikipedia.org/wiki/Cache_replacement_policies).

To make the write-through caching, on `hearty-store-put`, you just need to write it into two locations: the local cache and the remote server.

### Step #2 - Write-Back Caching
To make the write-through caching, on `hearty-store-put`, you just need to write it only into the local cache. However, you need to mark the cached file as **dirty** since, eventually, you need to update the dirty cached file back into the remote server.

Once the dirty cached file gets replaced by the others from the cache replacement policy, the client needs to send the cached file back to the server and the server must update the store instance with the accordingly.

### Step #3 - Cache Coherency Protocol
Since each client has its own write-back caching area, it is possible that the same file is cached in multiple clients. The issue is that the same file can be updated independently by each client. This will make each client see the same file differently. This inconsistency is unpredictable. You at least need to define the protocol to make all the caching areas coherent.

The protocol you will use in this assessment is based on the protocol in *Sprite*. (You may use AFSv2 and NFSv3 we discussed in class.) The idea of Sprite is that you need to make sure that *all the reads will see the most recent write*. You can ensure this by disabling caching for the file whenever there are more than one clients modifying this file. In other words, if only one client is modifying the file, we allow caching it. However, **if there is one more client wants to modify the file, the client caching this file needs to flush the file back immediately.** The server will be the one managing this protocol.

## Task #3 - Ensure Internal Consistency (30 points)
You can see that the following operations consist of multiple steps:
- `hearty-store-init`: you at least need to create the store instance first then add the store into the server metadata.
- `hearty-store-destroy`: you at least need to remove the store from the server metadata first and then destroy the store instance.
- `hearty-store-put`: you at least need to put the file first then put the entry `(unique identifier, file location)` to the server metadata.

Even though you can ensure internal consistency by using **update ordering**, you will need to do the cleanup process.

Instead, in this assessment, you will need to ensure internal consistency of the **server metadata** by implementing a real atomicity mechanism, [*Write-Ahead Logging (WAL)*](https://martinfowler.com/articles/patterns-of-distributed-systems/write-ahead-log.html). You need to ensure that the log must be in the persistent storage (i.e., on disk).

To implement WAL, you need to create a persistent log where you log all the steps you did in the operation in. **Most importantly, you need to log before doing each step.** After your operation ends, you need to commit the log so that you know that the operation is actually end. To commit the log, you just need to append the **COMMIT** message to the log.

Once your server fails, you can revisit the log. If the last log is the **COMMIT** message, you do not need to do anything. However, if the last log is not the **COMMIT** message, then the operation is not entirely done. You need to revert the log until you hit the nearest the **COMMIT** message or the beginning of the log.

There is an issue that could happen. What if the server dies after logging the step but before doing the step? Therefore, you need to find the way to verify whether this step has been done or not based on the log. For example, if I log that I "wrote to block 5." I will need to check whether block 5 has been written or not. If not, I do not need to revert this log.

### Implementing Write-Ahead Logging (WAL)

For simplicity, you only need to ensure atomicity for `hearty-store-put`. Basically, `hearty-store-put` should contain the following steps:

1) [Write] Update the store metadata to say that this block is not available
2) [Write] Put the file into an available block
3) [Write] Generate an identifier (`id`) and put a file entry into the store metadata (could be a pair of `(id, block_index, size)`) (Depends on how you generate `id`)

You can see that there will be 2 or 3 steps for doing `hearty-store-put`.

For the first step, you can write the log as `allocate [block_index]`. To verify, you just need to check whether the metadata says whether the `[block_index]` is allocated or not. To revert, you need to set the `[block_index]` to be unallocated.

For the second step, you can write the log as `put file with [checksum] into [block_index] replacing [old_block_data]`. You can use [md5](https://docs.openssl.org/master/man3/MD5/) to do checksum. The reason why we do `[checksum]` is to verify. We can verify whether the write has been done by reading the file and matching with the `[checksum]`. To revert, basically, we just need to replace the block with the [old_data_block].

For the third step, you may write the log as `add entry [id] [block_index] [size] from [old_id] [old_block_index] [old_size]`. To verify, you just need to read the metadata. To revert, you need to use the old data part.

**Since your implementation does not need to be the same as mine, you just need to use the concept of logging, verifying, and reverting to implement yours.**

## Code Style
You should follow a good coding convention. In this class, please stick with the CMU 15-213's Code Style.

https://www.cs.cmu.edu/afs/cs/academic/class/15213-f24/www/codeStyle.html

## Programming Language
You can use C, C++, Rust to implement this project. Other programming languages rather than these must be approved by the instructor first. (But, definitely, Python cannot)

## Submission
Please put all of your code and **instructions to run the code** in the `/`. You should describe a bit on your code structure so that it is easy for me to navigate through your code when grading. All of the instructions and descriptions should be in `/DESCRIPTION.md`.