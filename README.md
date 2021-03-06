<p align="center">
<img 
    src="resources/logo.png" 
    width="350" height="85" border="0" alt="SummitDB">
</p>

SummitDB is an in-memory, [NoSQL](https://en.wikipedia.org/wiki/NoSQL) key/value database. It persists to disk, uses the [Raft](https://raft.github.io/) consensus algorithm, is [ACID](https://en.wikipedia.org/wiki/ACID) compliant, and built on a transactional and strongly-consistent model. It supports [custom indexes](https://github.com/tidwall/summitdb/wiki/SETINDEX), [geospatial data](https://github.com/tidwall/summitdb/wiki/SETINDEX#spatial), [JSON documents](https://github.com/tidwall/summitdb/wiki/SETINDEX#json), and [user-defined JS scripting](https://github.com/tidwall/summitdb/wiki/EVAL).

Under the hood it utilizes [Finn](https://github.com/tidwall/finn), [Redcon](https://github.com/tidwall/redcon), [BuntDB](https://github.com/tidwall/buntdb), [GJSON](https://github.com/tidwall/gjson), and [Otto](https://github.com/robertkrimen/otto).

## Features

The goal was to create a fast data store that provides:

- [In-memory with disk persistence](#in-memory-disk-persistence)
- [Strong-consistency and durability](#consistency-and-durability)
- High-availability
- Simplified Redis-style APIs
- Ordered key space
- [Indexing on values](https://github.com/tidwall/summitdb/wiki/SETINDEX)
- [JSON documents](#json-indexes)
- [Spatial indexing](https://github.com/tidwall/summitdb/wiki/SETINDEX#spatial)


## Getting started

### Building SummitDB

SummitDB can be compiled and used on Linux, OSX, Windows, FreeBSD, and probably others since the codebase is 100% Go. We support both 32 bit and 64 bit systems. Go must be installed on the build machine.

To build simply:

```
$ make
```

It's a good idea to install the [redis-cli](http://redis.io/topics/rediscli).

```
$ make redis-cli
```

To run tests:

````
$ make test
```

### Running

First start a single-member cluster:
```
$ ./summitdb-server
```

This will start the server listening on port 7481 for client and server-to-server communication.

Next, let's set a single key, and then retrieve it:

```
$ ./redis-cli -p 7481 SET mykey "my value"
OK
$ ./redis-cli -p 7481 GET mykey
"my value"
```

Adding members:
```
$ ./summitdb-server -p 7482 -dir data2 -join :7481
$ ./summitdb-server -p 7483 -dir data3 -join :7481
```

That's it. Now if node1 goes down, node2 and node3 will continue to operate.

## Difference between SummitDB and Redis

*It may be worth noting that SummitDB is not a Redis clone. Redis has a lot of commands and data types that are not available in SummitDB, such Sets, Hashes, Sorted Sets, and PubSub.*

- **Ordered key space** - SummitDB provides one key space that is a large B-tree. An ordered key space allows for stable paging through keys using the [KEYS](https://github.com/tidwall/summitdb/wiki/KEYS) command. Redis uses an unordered dictionary structure and provides a specialized [SCAN](http://redis.io/commands/scan) command for iterating through keys.
- **Everything a string** - SummitDB stores only strings which are exact binary representations of what the user stores. Redis has many [internal data types](http://redis.io/topics/data-types-intro), such as strings, hashes, floats, sets, etc. 
- **Raft clusters** - SummitDB uses the Raft consensus algorithm to provide high-availablity. Redis provides [Master/Slave replication](http://redis.io/topics/replication). 
- **Javascript** - SummitDB uses Javascript for user-defined scripts. Redis uses Lua.
- **Indexes** - SummitDB provides an API for indexing the key space. Indexes allow for quickly querying and iterating on values. Redis has specialized data types like Sorted Sets and Hashes which can provide [secondary indexing](http://redis.io/topics/indexes).
- **Spatial indexes** - SummitDB provides the ability to create spatial indexes. A spatial index uses an R-tree under the hood, and each index can be up to 20 dimensions. This is useful for geospatial, statistical, time, and range data. Redis has the [GEO API](http://redis.io/commands/geoadd) which allows for using storing and querying geospatial data using the [Geohashes](https://en.wikipedia.org/wiki/Geohash).
- **JSON documents** - SummitDB allows for storing JSON documents and indexing fields directly. Redis has Hashes and a JSON parser via Lua.

<a name="in-memory-disk-persistence"></a>
## In-memory with disk persistence
SummitDB store all data in memory. Yet each writable command is appended to a file that is used to rebuild the database if the database needs to be restarted. 

This is similar to [Redis AOF persistence](http://redis.io/topics/persistence).


## JSON Indexes

Indexes can be created on individual fields inside JSON documents.

For example, let's say you have the following documents:

```json
{"name":{"first":"Tom","last":"Johnson"},"age":38}
{"name":{"first":"Janet","last":"Prichard"},"age":47}
{"name":{"first":"Carol","last":"Anderson"},"age":52}
{"name":{"first":"Alan","last":"Cooper"},"age":28}
```

Create an index:

```
> SETINDEX last_name user:* JSON name.last
```

Then add some JSON:
```
> SET user:1 '{"name":{"first":"Tom","last":"Johnson"},"age":38}'
> SET user:2 '{"name":{"first":"Janet","last":"Prichard"},"age":47}'
> SET user:3 '{"name":{"first":"Carol","last":"Anderson"},"age":52}'
> SET user:4 '{"name":{"first":"Alan","last":"Cooper"},"age":28}'
```

Query with the ITER command:

```
> ITER last_name
1) "user:3"
2) "{\"name\":{\"first\":\"Carol\",\"last\":\"Anderson\"},\"age\":52}"
3) "user:4"
4) "{\"name\":{\"first\":\"Alan\",\"last\":\"Cooper\"},\"age\":28}"
5) "user:1"
6) "{\"name\":{\"first\":\"Tom\",\"last\":\"Johnson\"},\"age\":38}"
7) "user:2"
8) "{\"name\":{\"first\":\"Janet\",\"last\":\"Prichard\"},\"age\":47}"
```

Or perhaps you want to index on age:

```
> SETINDEX age user:* JSON age
> ITER age
1) "user:4"
2) "{\"name\":{\"first\":\"Alan\",\"last\":\"Cooper\"},\"age\":28}"
3) "user:1"
4) "{\"name\":{\"first\":\"Tom\",\"last\":\"Johnson\"},\"age\":38}"
5) "user:2"
6) "{\"name\":{\"first\":\"Janet\",\"last\":\"Prichard\"},\"age\":47}"
7) "user:3"
8) "{\"name\":{\"first\":\"Carol\",\"last\":\"Anderson\"},\"age\":52}"
```

It's also possible to multi-index on two fields:

```
> SETINDEX last_name_age user:* JSON name.last JSON age
```

For full JSON indexing syntax check out the [SETINDEX](https://github.com/tidwall/summitdb/wiki/SETINDEX#json) and [ITER](https://github.com/tidwall/summitdb/wiki/ITER) commands.

<a href="raft-commands"></a>
Built-in Raft Commands
----------------------
Here are a few commands for monitoring and managing the cluster:

- **RAFTADDPEER addr**  
Adds a new member to the Raft cluster
- **RAFTREMOVEPEER addr**  
Removes an existing member
- **RAFTLEADER**  
Returns the Raft leader, if known
- **RAFTSNAPSHOT**  
Triggers a snapshot operation
- **RAFTSTATE**  
Returns the state of the node
- **RAFTSTATS**  
Returns information and statistics for the node and cluster

Consistency and Durability
--------------------------

SummitDB is tuned by design for strong consistency and durability. A server shutdown, power event, or `kill -9` will not corrupt the state of the cluster or lose data. 

All data persists to disk. SummitDB uses an append-only file format that stores for each command in exact order of execution. 
Each command consists of a one write and one fync. This provides excellent durability.

### Read Consistency

The `--consistency` param has the following options:

- `low` - all nodes accept reads, small risk of [stale](http://stackoverflow.com/questions/1563319/what-is-stale-state) data
- `medium` - only the leader accepts reads, itty-bitty risk of stale data during a leadership change
- `high` - only the leader accepts reads, the raft log index is incremented to guaranteeing no stale data. **this is the default**

For example, setting the following options:

```
$ summitdb --consistency high
```

Provides the highest level of consistency. The default is **high**.


Leadership Changes
------------------

In a Raft cluster only the leader can apply commands. If a command is attempted on a follower you will be presented with the response:

```
> SET x y
-TRY 127.0.0.1:7481
```

This means you should try the same command at the specified address.

Commands
--------

Below is the complete list of commands and documentation for each.

**Keys and values**  
[APPEND](https://github.com/tidwall/summitdb/wiki/APPEND), 
[BITCOUNT](https://github.com/tidwall/summitdb/wiki/BITCOUNT), 
[BITOP](https://github.com/tidwall/summitdb/wiki/BITOP), 
[BITPOS](https://github.com/tidwall/summitdb/wiki/BITPOS), 
[DBSIZE](https://github.com/tidwall/summitdb/wiki/DBSIZE),
[DECR](https://github.com/tidwall/summitdb/wiki/DECR), 
[DECRBY](https://github.com/tidwall/summitdb/wiki/DECRBY), 
[DEL](https://github.com/tidwall/summitdb/wiki/DEL),
[EXISTS](https://github.com/tidwall/summitdb/wiki/EXISTS),
[EXPIRE](https://github.com/tidwall/summitdb/wiki/EXPIRE),
[EXPIREAT](https://github.com/tidwall/summitdb/wiki/EXPIREAT),
[FLUSHDB](https://github.com/tidwall/summitdb/wiki/FLUSHDB),
[GET](https://github.com/tidwall/summitdb/wiki/GET), 
[GETBIT](https://github.com/tidwall/summitdb/wiki/GETBIT), 
[GETRANGE](https://github.com/tidwall/summitdb/wiki/GETRANGE), 
[GETSET](https://github.com/tidwall/summitdb/wiki/GETSET), 
[INCR](https://github.com/tidwall/summitdb/wiki/INCR), 
[INCRBY](https://github.com/tidwall/summitdb/wiki/INCRBY), 
[INCRBYFLOAT](https://github.com/tidwall/summitdb/wiki/INCRBYFLOAT), 
[KEYS](https://github.com/tidwall/summitdb/wiki/KEYS),
[MGET](https://github.com/tidwall/summitdb/wiki/MGET), 
[MSET](https://github.com/tidwall/summitdb/wiki/MSET), 
[MSETNX](https://github.com/tidwall/summitdb/wiki/MSETNX), 
[PDEL](https://github.com/tidwall/summitdb/wiki/PDEL),
[PERSIST](https://github.com/tidwall/summitdb/wiki/PERSIST),
[PEXPIRE](https://github.com/tidwall/summitdb/wiki/PEXPIRE),
[PEXPIREAT](https://github.com/tidwall/summitdb/wiki/PEXPIREAT),
[PTTL](https://github.com/tidwall/summitdb/wiki/PTTL),
[RENAME](https://github.com/tidwall/summitdb/wiki/RENAME),
[RENAMENX](https://github.com/tidwall/summitdb/wiki/RENAMENX),
[SET](https://github.com/tidwall/summitdb/wiki/SET), 
[SETBIT](https://github.com/tidwall/summitdb/wiki/SETBIT), 
[SETRANGE](https://github.com/tidwall/summitdb/wiki/SETRANGE), 
[STRLEN](https://github.com/tidwall/summitdb/wiki/STRLEN),
[TTL](https://github.com/tidwall/summitdb/wiki/TTL)

**Indexes and iteration**  
[DELINDEX](https://github.com/tidwall/summitdb/wiki/DELINDEX),
[INDEXES](https://github.com/tidwall/summitdb/wiki/INDEXES),
[ITER](https://github.com/tidwall/summitdb/wiki/ITER),
[RECT](https://github.com/tidwall/summitdb/wiki/RECT),
[SETINDEX](https://github.com/tidwall/summitdb/wiki/SETINDEX)

**Transactions**  
[MULTI](https://github.com/tidwall/summitdb/wiki/MULTI),
[EXEC](https://github.com/tidwall/summitdb/wiki/EXEC),
[DISCARD](https://github.com/tidwall/summitdb/wiki/DISCARD)

**Scripts**  
[EVAL](https://github.com/tidwall/summitdb/wiki/EVAL),
[EVALRO](https://github.com/tidwall/summitdb/wiki/EVALRO),
[EVALSHA](https://github.com/tidwall/summitdb/wiki/EVALSHA),
[EVALSHARO](https://github.com/tidwall/summitdb/wiki/EVALSHARO),
[SCRIPT LOAD](https://github.com/tidwall/summitdb/wiki/SCRIPT-LOAD),
[SCRIPT FLUSH](https://github.com/tidwall/summitdb/wiki/SCRIPT-FLUSH)

**Raft management**  
[RAFTADDPEER](https://github.com/tidwall/summitdb/wiki/RAFTADDPEER),
[RAFTREMOVEPEER](https://github.com/tidwall/summitdb/wiki/RAFTREMOVEPEER),
[RAFTLEADER](https://github.com/tidwall/summitdb/wiki/RAFTLEADER),
[RAFTSNAPSHOT](https://github.com/tidwall/summitdb/wiki/RAFTSNAPSHOT),
[RAFTSTATE](https://github.com/tidwall/summitdb/wiki/RAFTSTATE),
[RAFTSTATS](https://github.com/tidwall/summitdb/wiki/RAFTSTATS)

## Contact
Josh Baker [@tidwall](http://twitter.com/tidwall)

## License

SummitDB source code is available under the MIT [License](/LICENSE).



