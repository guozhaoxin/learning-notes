This file used to record  Google file system, and the origin PDF is from https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/gfs-sosp2003.pdf



### Assumptions

In this article, the author gave some assumptions:

- the system is built from many cheap commodity Linux machines while component failures are norm rather than the exception;
- the system stores a very large number of large files, even multi-GB files are common; small files are also supported, but need not optimize for them;
- the workloads consist of two kinds of reads: large streaming reads and small random reads;
- the workloads have many large, sequential writes that append data to files; once written, files are seldom modified again;
- the system implements well-defined semantics for multiple clients that concurrently append to the same file; many producers may concurrently append to a file, so atomicity with minimal synchronization overhead is essential;
- high sustained bandwidth is important;



### Architecture

A GFS cluster consists a single master and many chunkservers, and chunkservers are used to store file data and response to requests from many clients for file read. Files are divided into fixed-size chunks, and each chunk is identified by an immutable and globally unique 64-bit chunk handle assigned by the master at the time  of chunk creation. Each chunk is replicated to more  than 1 machine for reliability.

The master is used to maintain all file system metadata, including namespace, access control information, map from files to chunks, and the current locations of chunks.  Besides, it also controls system-wide activities such as chunk lease management, gc of orphaned chunks, and chunk migration between chunkservers. The master periodically communicates with each chunkserver in heartbeat messages.

Clients interact with the master for metadata operations, and then goes directly to the chunkservers, the master does not contain any file data. The clients and chunkservers neither caches any file data to simplify the system and eliminate the cache coherence issue. Howerver, the clients can cache metadata. 