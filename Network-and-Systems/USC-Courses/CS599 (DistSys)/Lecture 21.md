**Paper:** [FAWN: A Fast Array of Wimpy Nodes]([FAWN: A Fast Array of Wimpy Nodes](https://www.sigops.org/s/conferences/sosp/2009/papers/andersen-sosp09.pdf))

The goal of the system de jour is make a distributed system that is performant, but also efficient. In this case, we want to just make KV-store that does not need to hoover up power to keep its DRAM alive.

# Introduction

Any distributed system relies on a whole series of servers to operate. Today, most of them need an entire data center worth of servers to work. When we look at the *Total Cost of Ownership* (TCO) for any set of servers, we see multiple aspects:

- Real estate cost
- Hardware cost
- Maintenance cost
- **Power and cooling**, by far the biggest one in the long term, especially now ...

The paper relies on some key observations to motivate the design of its system:

- Even when the paper was written (more than a decade ago), about half of the 3-year TCO of a data center is just for power and cooling.
- Most systems are IO bound, not compute bound, so they don't need top of the line CPUs
- Fast CPUs consume a huge amount of power, so idle CPUs are a big hit ...
  The cost of this is comprised of the following (among others):
	- Maintaining low level and high level caches needs lots of energy
	- Many a times these CPUs can be overclocked, which disproportionately increases their power consumption but provides better throughput on a single core.
	- CPUs can run on a low power state where they run 'slower', but that is not slow enough in many scenarios
- DRAM consumes a surprisingly large amount of power. When the paper was written, 2 GB of DRAM needed around the same power as a 1 TB disk.

So here is the key insight, *Flash Storage* exists:
- low power
- can provide high storage
- lasts reasonably long (assuming a good file system is used ...)
## Flash Storage

Flash storage (in this case, NAND flash storage, or SSDs), are very different compared to DRAM and spinning disks:
- Data is stored by charging MOSFETs or draining them.
- Can read and write individual pages (e.g. 2 - 4 KBs)
- Can only provide a finite number of write cycles per MOSFET (around 1000 - 100000)
	- To rectify this, many SSDs come with a Flash Translation Layer (FTL) that spreads write out by default
- <u>(THE BIG ONE)</u> If you want to write to an SSD block, you MUST erase all of it first (a block being 32-64 pages normally). This is terrible for random writes.

To rectify the final limitation, many systems that implement file systems on SSDs, will pay for extra complexity and use a *Log Structured File System* (see CS 655, Advanced OS). 

# FAWN Design

![[Pasted image 20241119123319.png]]

The paper describes the design of a Key-Value store (NOT a database, the reason is that the system that was implemented does not provide any transactional semantics).

- The ***frontend*** is a series of rather strong machines that serve client requests. This part can be scaled up or down as much as needed.
	- Frontends are assigned a range of hashes to serve (for 3 frontends, each one gets a third of the hash ring to serve). These ranges are assigned through a management node (that is also usually part of the frontend).
	- A frontend that receives a request outside of its range can forward the request to the correct frontend.
		- The reason we do this is because frontends maintain caches, and we want to make sure that the cache for the data in a particular range, remains in the associated frontend.
		- You don't need virtual nodes for frontends. Since even if the data gets shuffled in the backend, you can just flush the frontend caches (remember, frontend stores no data)
- The ***backend*** consists of a series of FAWN nodes that are connected through a backbone of Ethernet switches.
	- Consistent hashing is used to distribute data among nodes. To further load-balance the system, each physical node maintains multiple (about  5) *virtual nodes* that participate in communication and share data more evenly. Thus, each wimpy node maintains up to 5 non-contiguous hash ranges.
	- Each node maintains an *in-memory* hash table that tells where the data for each key is located.

The join-leave procedure is signaled by the wimpy nodes, the frontends recalculate and reassign wedges when configuration changes. Again, since they do not store data, when in doubt, just flush the cache.

So here is a power breakdown:
- The ring consists of wimpy nodes that consume around 3 watts when idle and 6 when busy.
- A much less wimpy frontend node serves requests constantly, so that consumes a lot more, around 50-100 watts.
- The ring is networked with a 16 port Ethernet switch that also serves the frontends. Those consume moderate power, typical a few tens of a watt.
## Lookups

![[Pasted image 20241119130600.png]]

Lookups need to be served quickly, thus we maintain any lookup table for a wimpy node on its limited, but still fast DRAM. The goal is to map some key to an offset on the append-only log that the file system for that node maintains.

There is again a problem here though. The DRAM here is truly limited (the hardware used for evaluation has just 256 MB), and of course it is needed for OS processes and management as well. Thus, we should make the lookup table pretty small.

FAWN makes a tradeoff here. Instead of using the whole key to map the indices, we chunk the key into two parts, and only a particular fraction of the key is used for lookup in the table (and thus actually kept there in the DRAM).
Of course, this means that there can be collision, so, FAWN stores the actual key and the value associated with it on the log entry that is pointed to by the table. Thus, when we actually query something and jump to the offset, we also need to verify that the key matches with what we wanted.

**Note:** Maybe I am stupid, but I just do not understand how the lookup works based on the paper, so some of the material that follows comes from the [source code]([GitHub - vrv/FAWN-KV: A Distributed Key-Value Store for FAWN](https://github.com/vrv/FAWN-KV). In particular, [this function](https://github.com/vrv/FAWN-KV/blob/cbe33711d022530f821079f9ff6b541cb5b52ecd/fawnds/fawnds.cc#L501).

First off, the datastore is designed to be tunable, meaning that we should be able to just say how many objects we want to store, and be able to create a hash table that can fit that size.

Thus, we get a hold of the maximum number of objects that we want to store, and with that, our hash table is created:
```c++
ds->hash_table_ =
	(struct HashEntry*) malloc(sizeof(struct HashEntry) * max_entries);
if (ds->hash_table_ == NULL) {
    perror("could not malloc hash table file\n");
    delete ds;
    return NULL;
}

// zero out the buffer
memset(ds->hash_table_, 0, sizeof(struct HashEntry) * max_entries);
```
Where a Hash Entry (referred to as a Hash Bucket confusingly in the paper) is defined as:
```cpp
/*
  Hash Entry Format
  D = Is slot deleted: 1 means deleted, 0 means not deleted.
  Needed for lazy deletion
  V = Is slot empty: 0 means empty, 1 means taken
  K = Key fragment
  O = Offset bits
  ________________________________________________
  |DVKKKKKKKKKKKKKKOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO|
  ¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯
*/
struct HashEntry {
	uint16_t present_key;
	uint32_t offset;
} __attribute__((__packed__));
```
Which means that each entry is a total of 6 bytes (as mentioned in the paper), and the table in general needs `max_enties * 6` bytes to be `malloc`'d for it. The size of the hash table (i.e. the number of entries) is always set to be a power of 2, so that `hash_table_size - 1` would be an all 1 mask.

As such, for example, inserting an object with a given key would look like the following:
```c++
template <typename T>
int32_t FawnDS<T>::
FindHashIndexInsert(const char* key, uint32_t key_len, bool* newObj)
{
	int32_t hash_index = 0;
	for (
		u_int hash_fn_index = 0;
		hash_fn_index < HASH_COUNT;
		++hash_fn_index)
	{
		uint32_t indexkey = (*(Hashes::hashes[hash_fn_index]))(key, key_len);
		uint32_t hash_index_start = indexkey & (header_->hashtable_size-1);
		if (hash_fn_index == 2)
			print_payload((const u_char*) &indexkey, sizeof(indexkey));
		// PROBES_BEFORE_REHASH = 8
		// Below sets the first 3 bits to 0 ...
		hash_index_start &= (~(PROBES_BEFORE_REHASH-1));

		for (
			hash_index = hash_index_start;
			(uint32_t)hash_index < hash_index_start + PROBES_BEFORE_REHASH;
			++hash_index)
		{
			uint16_t vkey = verifykey(hash_index);
			// Find first open spot or find the same key
			// `exists` means be valid and not deleted
			if (!exists(hash_index)) {
				// This is a new object, not an
				// updated object for an existing key.
				*newObj = true;
				return hash_index;
			} else if (vkey == keyfragment_from_key(key, key_len)) {
				off_t datapos = hash_table_[hash_index].offset;
				DataHeader data_header;
				string ckey;
				if (
					datastore->ReadIntoHeader(datapos, data_header, ckey) &&
					ckey.length() == key_len &&
					memcmp(ckey.data(), key, key_len) == 0
				)
					return hash_index;
			}
		}
	}
	return -1;
}
```
In plain English, when inserting an object:
- Sequentially invoke that hash functions on the given key (each hash gives a 32 bit value back). This result is the `indexKey`, and we can mask it with an all 1 bit string of appropriate size to make fit within the hash table.
  This is that `hash_table_index`, and the *Index* that the paper actually refers to.
- So in the implementation, we can retry and increment entries up to 8 times before moving on to the next hash function.
- Use the hash index found from above, get the associated entry in the array of hashes, and check if it `exists`, meaning that it is both valid and not deleted (in the paper, validity just means not deleted, so there is only 1 bit instead of 2 to consider ...)
	- If it does not exist, return this index, as it is appropriate for this new object.
	- If it does exist:
		- Get the key fragment. In the above, this is referred to as the `vkey`, which we get by looking up the entry associated with this index, getting the key field and then masking it.
		- If the above is equal to the key fragment part of the original key, then read the object from the log and just compare the keys. If they are equal, then this object already exists, don't insert it.
		- If the above didn't happen, congrats, you got a collision! Either probe the next entry, you change the hash function.

As such, in the worst case, we would have 24 iterations before giving up (3 hash functions, 8 probes), and also 24 queries of the SSD at worst before giving up, so the claim that it only accesses 2 times is just wrong or the paper just means that the probability of having 2 SSD reads is that much ...

Below is the pseudocode for lookups:

![[Pasted image 20241119132359.png|500]]

This is the actual code:
```c++
template <typename T>
bool FawnDS<T>::Get(const char* key, uint32_t key_len, string &data) const
{
	if (key == NULL)
		return false;

	// use DataHeaderExtended for app readahead
	int32_t hash_index = 0;

	for (
		u_int hash_fn_index = 0;
		hash_fn_index < HASH_COUNT;
		hash_fn_index++)
	{
		uint32_t indexkey = (*(Hashes::hashes[hash_fn_index]))(key, key_len);
		uint32_t hash_index_start = indexkey & (header_->hashtable_size-1);
		hash_index_start &= (~(PROBES_BEFORE_REHASH-1));

		for (
			hash_index = hash_index_start;
			(uint32_t)hash_index < hash_index_start + PROBES_BEFORE_REHASH;
			++hash_index)
		{
			uint16_t vkey = verifykey(hash_index);
			if (!valid(hash_index)) {
				return false;
			}
			else if (deleted(hash_index)) {
				continue;
			}
			else if (vkey == keyfragment_from_key(key, key_len)) {
				off_t datapos = hash_table_[hash_index].offset;
				if (datastore->Read(key, key_len, datapos, data))
					return true;
			}
		}

	}
	return false;
}
```
For a $k$ bit key fragment, and $i$ bit index, we would have $2^i$ entries in our hash table. However, the claim that we only need to go to the SSD 1 in whatever value just isn't accurate, and the implementation does not show that at all.
## Evaluation

![[Pasted image 20241203093001.png|400]]

In the above, remember that the DRAM size is 256 MB, thus when we approach that, FAWN actually needs to keep hash entries in SSDs, which slows it down. This explains the sudden drop of throughput from 125 to 256 MB.

![[Pasted image 20241119134335.png|500]]

In the above, SSDs in general work well with semi-random read/writes, thus having less files would help performance. Since the LSF used for the backend needs only a single append, write heavy loads are a perfect match for this system.

![[Pasted image 20241119134503.png|500]]

Above is the benchmarking of a 21-node system.
