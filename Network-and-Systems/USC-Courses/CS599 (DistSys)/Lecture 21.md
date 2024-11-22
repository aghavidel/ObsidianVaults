**Paper:** FAWN: A Fast Array of Wimpy Nodes
The goal of the system de jour is make a distributed system that is performant, but also efficient. In this case, we want to just make KV-store.

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
  The cost of this is comprise of:
	- Maintaining low level and high level caches needs a lot of energy
	- Many a times these CPUs can be overclocked, which disproportionately increases their power consumption but provides better throughput on a single core.
	- CPUs can run on a low power state where they run 'slower', but that is not slow enough in many scenarios
- DRAM consumes a surprisingly large amount of power. When the paper was written, 2 GB of DRAM needed around the same power as a 1 TB disk.
- **Flash storage** on the other hand is:
	- low power
	- can provide high storage
	- lasts reasonably long (assuming a good file system is used ...)

## Flash Storage

Flash storage (in this case, NAND flash storage, or SSDs), are very different compared to DRAM and spinning disks:
- Data is stored by charging MOSFETs or draining them.
- Can read and write individual pages (e.g. 2 - 4 KBs)
- Can only provide a finite number of write cycles per MOSFET 
	- To rectify this, many SSDs come with a Flash Translation Layer (FTL) that spreads write out by default
- (THE BIG ONE) If you want to write to an SSD block, you MUST erase all of it first (a block being 32-64 pages normally). This is terrible for random writes.

To rectify the final limitation, many systems that implement file systems on SSDs, will pay for extra complexity and use a *Log Structured File System* (see CS 655, Advanced OS). 

# FAWN Design

![[Pasted image 20241119123319.png]]

The paper describes the design of a Key-Value store (NOT database, the reason is that the system that was implemented does not provide any transactional semantics).
- The frontend is a series of rather strong machines that serve client requests. This part can be scaled up or down as much as needed.
	- Frontends are assigned a range of hashes to serve (for 3 frontends, each one gets a third of the hash ring to serve). These ranges are assigned through a management node.
	- A frontend that receives a request outside of its range can forward the request to the correct frontend.
		- The reason we do this is because frontends maintain caches, and we want to make sure that the cache for the data in a particular range, remains in the associated frontend.
		- You don't need virtual nodes for frontends. Since even if the data gets shuffled in the backend, you can just flush the frontend caches (remember, frontend stores no data)
- The *backend* consists of a series of FAWN nodes that are connected through a backbone of Ethernet switches. 
	- Consistent hashing is used to distribute data among nodes. To further load-balance the system, each physical node maintains multiple (about  5) *virtual nodes* that participate in communication and share data more evenly.
	- Each node maintains an *in-memory* hash table that tells where the data for each key is located.

## Lookups

![[Pasted image 20241119130600.png]]

For lookups, we use 160-bit keys:
- 16 lower bits are used for indexing. This needs to be small, since we only have around a few GB of memory to keep the hash table, and so this index space needs to be limited.
- Next 15 bits are the *Key Fragment* ....
(I DO NOT UNDERSTAND HOW THIS WORKS, READ THE PAPER!)

![[Pasted image 20241119132359.png|500]]

## Evaluation

(IN THE SLIDES, THE DROP FROM 125 TO 250 IS BECAUSE DATA NO LONGER FITS IN MEMORY)

![[Pasted image 20241119134335.png|500]]

SSDs in general work well with semi-random read/writes, thus having less files would help performance. Since the LSF used for the backend needs only a single append, write heavy loads are a perfect match for this system.

![[Pasted image 20241119134503.png|500]]

Above is the benchmarking of a 21-node system. However, 