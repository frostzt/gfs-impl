# My Notes on the Paper

## Design Assumptions

- Built using commodity hardware means failures are common,
therefore monitor itself and recover.
- Aiming for Millions of files and multi-GB files small files
are there too but we won't optimize for them.
- Focus is more towards large streaming reads, these reads are
over contigous region of files. Also have random reads but we
prefer optimizing and sorting them before we perform such.
- Large sequential writes are to be expected that append data
however files once written aren't modified (frequently) however
small writes at arbitrary positions are supported.
- Multiple clients can concurrently append to the same file.
- Files are often used as producer-consumer queues or for
many-way merging (k-way merging)
- Hundrends of producers running one per machine will
concurrently append to a file. Atomicity with minimal sync
overhead is needed.
- Consumers can simultaneously read through a file that is
being written.
- High sustained bandwidth > low latency

## Interface

Generic supported:

- create
- delete
- open
- close
- read
- write
- snapshot -- creates copy of a file or a directory tree at low cost
- record append -- allow multiple clients to append concurrently guarentees atomicity

## Architecture

- Consists of a single `master` and multiple `chunkservers` and is accessed by
multiple `clients`
- Files are divided into fixed-size `chunks`
- Each chunk can be identified by a 64-bit `chunkhandle` assigned by the master
at the time of chunk creation.
- `chunkservers` store these chunks as linux files and read/write chunk data by
a `chunkhandle` and byte range.
- Each file is replicated on multiple `chunkservers`, default is 3 replicas but
can be configured by the user.
- The master maintains all filesystem metadata, like, namespace, access control
info, mappings from files to chunks, and current locations of chunk.
- Master also controls system wide activities, like, chunk lease management,
garbage collection of orphaned chunks, and chunk migration between `chunkservers`
- Master periodically communicates with each `chunkservers` in HeartBeat messages
to give it instructions and collect its state.
- Client code linked into each application implements the filesystem API

![GFS Diagram](/docs/images/GFS%20Diagram.png)

- A single master simplifies the design, a global knowledge makes it easy for chunk
placements, replication, and any other stuff. However its usage must be minimized
as it can become a bottleneck.
- Clients will NOT write data through master, they'll ask master for info which `chunkservers`
to contact to and cache that info and write into that `chunkserver`

## Flow

- Given that every file is divided into equal size chunks (64 MB = ~67,108,864 bytes)
and we want data from let's say 100,000,000 bytes as offset
- The client can calculate `chunkindex` as:

```plaintext
chunkindex = 100,000,000 / 67,108,864
```

The above gives us `1` as a remainder which tells us that the data is in chunk 1

- The client calls the master with the `chunkindex` and the `filename`
- The master replies with `chunkhandle` and locations of replicas
- The client caches this key and further client-master interactions are no longer
needed.

Its most likely that the data that a consumer might want is the next node or
contiguous, therefore the master can return metadata for chunks next to the
current chunk further reducing the need of asking the master.

## Key Design Parameters, Assumptions and whatever else

### Single Master

Cleared this above as to why this is needed.

### Chunk Size

- Chosen size is 64 MB
- Each chunk replica is stored as a plain linux file
- Given the chunk size is large (64 MB), Lazy Space Allocation avoid wasting space.
- The advantages of choosing large chunk size are:
  - Large chunk size means less reads as requesting it once gets us most data
  - Even for small reads its easy as the client can cache and get data which might
  already be present.
  - With larger chunks since the client is likely to perform more ops on the same
  chunk we can reduce network overhead by keeping a persistant TCP connection.
  - This also results in a smaller metadata size and this metadata now can be kept
  in memory which provides further benefits.
- There are disadvantages too:
  - Hotspots can become common as for small files chunks can be just one
  - FWIW one way to solve this is by increasing the replication factor

## Metadata

Uses three major types of metadata that we'll store:

- The file and chunk namespaces
- Mapping from files to chunks
- Locations of each chunks replicas

The file, chunk namespaces and the mapping from file to chunks is also kept persistent
by logging mutations to an `operator log` stored on the master and replicated
to remote.

- The master never stores all the chunk locations rather it asks the `chunkservers`
for the locations and details on master start and when a new `chunkserver` joins
the cluster.

### In-Memory Data Structures

- Given metadata is stored in memory its easy for the master to scan it periodically
- The periodic scan is used to implement:
  - Chunk Garbage Collection
  - Re-replication in case of chunkserver failure
  - Chunk migration to balance load and disk space

### Chunk Locations

- The master doesn't keep everything in itself rather it asks the
Chunkservers on startup
- Given the above is rather more efficient we will keep asking the `chunkservers`
rather than storing metadata fully in master as `chunkservers` going down and
rejoining the cluster

### Operation Log

- Historical records of all critical metadata changes
- Master will batch several log records thereby reducing impact of saving,
replication, and flush
- Master recovers its state by replaying the operation log
- The idea is that the log should be kept small so the master will keep
taking checkpoints and then if a crash happens it will first restore from
the local disk and then replay the rest of the log.
- The checkpoint is in a compact B-tree like form that can be directly mapped
into memory and used for namespace lookup without extra parsing.
- The master switches to a new log file and creates the new checkpoint in
a separate thread. The new checkpoint includes all mutations before the switch.
