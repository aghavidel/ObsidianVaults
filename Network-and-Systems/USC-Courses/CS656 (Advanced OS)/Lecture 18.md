# Multi-Version File System

Until now, we have discussed multiple different ways that a user can lose data in a file system. Like:
- Bit errors
- Disk failures
- Machine restart
- Power outage

But now, we are going to look into a much more nefarious source, **User errors!**
These can be wrongly deleting a file, or doing erroneous modifications to a file or at a wrong time. So how to fix these?

Now, a strawman approach would be to use a *Recycle Bin*, and set the default action for file deletion to be "move it to the bin". Then, we can decide when to really delete things by emptying the trash.
This is not optimal since a user still needs to make sure to restore the file before trash is thrown out, and it is possible to get unlucky, and we cannot be to conservative, since the trash bin can get very large. The bigger problem however is that the recycle bin provides no protection for erroneous file modifications. 

A more brute force approach would be to *checkpoint the whole file system*. This works, but:
- It has extreme overhead
- It requires policing and garbage collection, which might be fine, but is still difficult
- Changes between checkpoints cannot be rolled-back correctly 
- There is not enough control on checkpointing time frame. Not every file needs to be checkpointed greedily, whereas for some files that change a lot, we might need a lot of checkpointing

## Elephant FS

The high level approach is to create a new version the moment a user updates/deletes a file.
This is feasible, most importantly because disks are becoming cheaper. The FS should allow the user to access a version of the file that existed in the past.

So the first challenge, how to minimize the overhead of this?

![[Pasted image 20240320143013.png|500]]

A naïve implementation would scale the checkpoint size based on the number of edits, but that is just too much. So we need to make things a bit better. One way would be to leverage copy-on-write. 
For every edit, we create a new *inode*, like the following:

![[Pasted image 20240320143244.png|500]]

We then add the changed blocks to the each inode and then *point* to unchanged versions. So for inode number 2 for example:
- We point to the first 3 gray block in inode number 1
- We point to a new green block specifically for inode 2
- We point to the rest of the gray blocks in inode 1

![[Pasted image 20240415055637.png|500]]

And that is it! With one assumption: **The user CANNOT edit the old versions**. They can rollback, and start editing on a fresh branch, but not on the previous branch. This is really is quite similar to how Git works.

We also need to minimize the space overhead by reducing the number of versions. So:
- Create only new version on file open. We accept the price that we cannot rollback partial edits (i.e. if 100 edits were made, but one of them was wrong,  we lost all of them)
- Skip read-only files or system files or any file that does not need a history
- Make backups less granular as we go further back

The user can also specify different levels of safety for each file. So for example, we can specify a file to be just kept as a single copy, or we can specify so that we can keep it "safe". A safe file should have a version from at least a particular while ago. So if the user specifies that we want to keep copies safe for 1 week, we can inspect different version. If $V_{i+1}$ is older than 1 week, then we can safely delete $V_i$ without a problem.

This prevents us losing data without knowing. What we DO NOT do is that we do not delete all copies older than a week, that is not correct.

## Differences With LFS

Of course in LFS, we cannot go back to previous versions. Previous versions are considered junk and cannot be reached from an inode very easily, so they will be garbage collected in the near future. The user also has no control on the garbage collector for LFS.

More importantly though, the cleaner for LFS is much heavier compared to this file system, since we do not need to write back live data or even detect live data.

There is also a comparison that can be made to Git, since file closes on the Elephant FS are exactly like Git commits. The difference mainly is that in Git, versions can be kept as a DAG, whereas in the Elephant FS, we can only keep linear versions, we cannot do forks or pull.

## Handling Deletions

For deletions, usually we first delete the inode, modify the directory that contained it, and then we add the blocks pointers to the free list. So in some way, this is another update operation, so why not treat it that way?

That turns out to be a bit more nuanced with this file system, since if you repeatedly create and delete a file, that I-Node will point to the same name, so the correct way would be to create a new I-Node for new files, but also note the creation and deletion time, and if needed, restore a previous version.

![[Pasted image 20240415063326.png|500]]

## Evaluation

![[Pasted image 20240415064347.png|500]]

First we do a micro-benchmark. We have 3 file systems to work with:
- **FFS**, you know this one
- **EFS-O** is the Elephant FS with no versioning
- **EFS-V** is the Elephant FS with versioning enabled

As you can see, most operations have the same speed, with the big difference only appearing on creation and deletion. 
- Creating a file is slower, since it requires updating the I-Node log and all the other stuff
- Deletion is *faster*, since it is done asynchronously as we mentioned

There is also a test where EFS was used to copy a large directory, and surprisingly, EFS is *faster*, being that it beats FFS. It is unclear exactly why, but it appears that EFS writes metadata asynchronously, which also isn't explained much :))) ..

![[Pasted image 20240415065729.png|400]]

As you can see (although note this is done in 1999), a huge amount of writes are directed only to Keep One files, which means EFS won't use too much space.
