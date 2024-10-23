# Transactions

**Paper:** [The Recovery Manager of the System R Database Manager](https://dl.acm.org/doi/pdf/10.1145/356842.356847)

The paper details the **System R** database management system. It was one of the most successful and complete implementations of a transactional semantic in a database.

## Background
Transactions, are defined as a series of actions that are atomic. Meaning that if nothing goes wrong, a transaction finishes completely without any other side effect, but in case something goes wrong, the system or the user can abort the transaction and return the system to exactly as it was before the transaction.

You should note that this is much more abstract than the atomic actions done by the OS, which are:
- Usually short and small
- Can be supported by the hardware
- Not necessarily data-oriented

Usually, these low-level operations are not supposed to *abort*, database transactions however can abort for many reasons. The mains ones:
1. Either the transaction itself aborts, since it fails a precondition on the data (this is what data-orientation means)
2. The transaction resources are not *available*, in practice this happens when the transaction does not manage to get the lock to one of it's resources (e.g. because another transaction is using it)
3. The system just gives up! A power outage, a crash, all sorts of unforeseen things.

"The SAVING/CHECKING examples goes here"

>[!REMINDER]
>In RAID, we also had a similar problem. If the system fails after committing but before updating disk parities, we can get screwed!

## Tools

### Shadowing

So how do we survive these failures?
- **Shadowing:** Do not update *in-place*! Create a copy of the data that is effected by the transaction, and once the update finishes, change the directory structure to point to the new data.  Keep the the modified data **unreachable** by the system until you are sure it is ready.

Is this safe?
1. **Failure During TXN:** Nothing gets destroyed, only the unreachable data block could be destroyed, and even then we don't have to worry about it, we just retry.
2. **Failure After TXN:** All is well! The disk persists the change and survives failure.
3. **Failure Before Pointer Switch** We have to retry the transaction from scratch, but otherwise, nothing is wrong.

As you can see, the moment where we change the pointer is when we are truly done with the transaction.

"See file system example .... note operations on a single sector are atomic by hardware support"

### Logging and Journaling

A *Log*, is a file that only supports read/append actions. Logs can journal the progress of a transaction with all details. 
The way it works is that one would log every step of the transaction and if a restart happens, we start by redoing everything that log said we should do. The most naive way of this would be start right from the beginning and redo all the operations. 
This of course can be very wasteful, since the log would have to contain the data result of each operation.

For example, if a transaction contains the operation to sort a file, the log would have to contain the sorted version of the file, which can be huge. It is much more natural to assume we just say "sort the file named X" in the log ...

The solution is to **combine** logging with shadowing. Do the operations of uncommitted transactions on copies of the data and point to it afterwards. 
The problem is that the log is still growing with every action, we need to invalidate some vents