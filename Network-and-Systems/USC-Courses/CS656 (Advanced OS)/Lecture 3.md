**Paper:** [RAID: High Performance, Reliable Secondary Storage

# Intro.

The paper discusses RAID, Redundant Array of Inexpensive Disks (also, Redundant Array of *Independent* Disks). Since the power of CPUs is growing much faster than the speed of which data can be transferred to/from a disk, most systems employ an *array* of disks, that allow parallel read/writes to these disks, effectively increasing the I/O speed by the replication rate.

This is all good, but the problem is that the more disks you have, the more they are likely to withstand a single failure, and the more likely you are to lose data.
Most applications cannot tolerate even a single altered bit of data, and as such, most systems will make use of Error Correction Codes (ECCs) to drastically lower the probability of data loss. We have pretty efficient ECCs, from Hamming Codes to Reed-Solomon codes, and recently with SSDs, we are beginning to use LDPCs to provide more headroom for their generally higher level of failure across the board.

