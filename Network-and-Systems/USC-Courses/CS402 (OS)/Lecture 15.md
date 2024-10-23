# Chapter 1 (Back Again!): I/O And File System

Let's recall IO redirection:
```c
if (fork() == 0) {
	close(1);
	close(2);
	if (open(".... /Output", "w") == -1)
		exit(-1)
	if (open(".... /Output", "w") == -1)
		exit(-1)
	execl(".... /prog", "program", 0);
	exit(-1);
}
```
In the above example, `stdout` and `stderr` are redirected to the same file, namely whatever `Output` is. Is this problematic?

Schematically, the process after doing `execl` will have a file table like the following:

![[Pasted image 20230328211649.png]]

First, remember that the cursor position on an opened file is part of it's context and thus maintained by the kernel. As such, if you write say, 100 bytes, to file descriptor 1, then the cursor for that entry will update to offset 100. However, if you write to file descriptor 2, you are *starting again from 0* and will thus overwrite whatever you wrote previously.

Is this bad? Well, if this isn't the behavior you wanted, then yes!
But probably not, so how would you fix this? The problem is that the file objects created by `open` are not shared, they are different. Can't we force the file descriptor table of the process to point to the same file object associated with `Output`? Like the following:

![[Pasted image 20230328212150.png]]

There is, with the `dup` Syscall that allocates a new file descriptor (the same way as before, so it picks the smallest unassociated file descriptor) tells it to point to the file object of a given file descriptor. Thus, if the first `open` call associates `Output` with file descriptor number 1, then `dup(1)` will associate the next free file descriptor (i.e. 2) with the file object for file descriptor 1!

Thus, when the cursor position updates, it updates for both! Thus the updated code would be:
```c
if (fork() == 0) {
	close(1);
	close(2);
	if (open(".... /Output", "w") == -1)
		exit(-1)
	dup(1);
	execl(".... /prog", "program", 0);
	exit(-1);
}
```
>[!IMPORTANT] File Table After `fork`
>When doing `fork`, the child will start with an exact copy of the file descriptor table of the it's parent. Thus, `fork`  actually updates the reference count!
>
>Thus, parent and child have *separate* file descriptor tables, but share the same *extended address space* indirectly. As such, using the same file in the code of the parent and child, ***can create a race condition***!
>
>Thus, for example, if you are trying to log events, you can't just let parent and child call the same function to write to the log file, as the behavior of this process would be unspecified.

## UNIX Pipe

To allow for redirecting IO between different processes without having to write code like the above, UNIX implements a data structure known as a *pipe*. 

![[Pasted image 20230328214210.png]]

Here, the sender essentially will have a file descriptor that it can write data on (the starting end of the pipe) and the receiver will have a file descriptor that it can read data from (the final end of the pipe). Thus, the pipe need only return to you the file descriptors for the start and end of the pipe. The pipe is actually *synchronized*, since it is implemented as a producer-consumer problem and thus is MUCH faster than locking mechanisms, since it uses Semaphores instead of Mutex.

Here is an example:
```c
int p[2]; // File descriptor pair of the pipe
pipe(p);
	// Now, p[0] refers to the read/output of the pipe
	// and p[1] refers to the write/input of the pipe
	
if (fork() == 0) {
	char buf[80];
	close(p[1]); // The child won't need the write end
	while (read(p[0], buf, 80) > 0) {
		// Process the data read from the pipe
	}
	exit(0); // Done!
}
else {
	char buf[80];
	close(p[0]); // The parent won't need the read end
	for (;;) {
		// Prepare data ...
		...
		write(p[1], buf, 80);
	}
}
```
Thus, the parent can now communicate with the child by writing data to the pipe and letting the child to read from it. We can create another pipe from the child to the parent as well to create a two-way communication!

Schematically, without the `close` calls, the file table looks like the following:

![[Pasted image 20230328215441.png]]

Once we `close` the ends that we don't need, the parent and child would have only a single way of communication, by writing to the pipe in the parent and reading from the pipe in the child.

>[!BUG] Common Bug
>If you don't close the unused end of a pip in the process, weird things can happen!

>[!NOTE] Pipe In UNIX Shell
>As you might remember, pipe has it's own syntax in the UNIX shell:
>```bash
>cat input_to_prog | ./prog
>```

>[!NOTE] Running In Background In UNIX Shell
>Calling any command, and ending it with an `&` sign, will signal to the shell that we want to run it in the background, and this means that the shell will *NOT* wait for the process associated with the command to die before giving you the shell prompt. 

## Random Access IO

It's possible to manipulate files by choosing explicitly the cursor location. This is called the *Seek* operation. This is done by:
- Choosing the cursor location by using the `lseek` Syscall.
- Write like before, but this starts from the new cursor position now.

The `lseek` Syscall accepts 3 arguments, the declaration being:
```c
off_t lseek(int fd, off_t offset, int whence);
```
Where:
- `fd` is a file descriptor
- `offset` is just an integer value which determines how much to change the cursor.
  The value of `offset` can be negative.
- `whence` determines *how* `offset` changes the current value of the cursor:
	- If `SEEK_SET`, `offset` would be measured from the **Beginning** of the file. Thus if we pass offset 100, we start writing from index 100.
	- If `SEEK_CUR`, the cursor will be set to the exact value of `offset`.
	- If `SEEK_END`, offset is measured from the end of the file.

>[!FAIL] Common Misconception
>The "End of File" isn't the last byte in the file, thus for file of length 100, the end of file isn't offset 99, it's the byte *right after it*, thus offset 100.

Here is an example:
```c
fd = open("some_file", O_RDONLY);
char buf[1];
fptr = lseek(fd, (off_t)(-1), SEEK_END);
while (fptr != -1) {
	read(fd, buf, 1);
	write(1, buf, 1);
	fptr = lseek(fd, (off_t)(-2), SEEK_CUR);
}
```
What is this?
Well, the first `lseek`, puts the cursor on the last byte of the file, in the loop, we:
- read a byte
- write it to `stdout`, increasing the cursor by 1
- use `lseek` to move the cursor back by 2 

Until `lseek` returns `-1`, saying that we went beyond the file. So what does this do?
The first loop prints offset 99, the second loop prints offset 98, and this continues until we print offset 0, afterwards the cursor will be 1 and calling `lseek` to set it back by 2, puts it on `-1` which is not valid and this the loop breaks now. 
This program prints the file content in reverse!

## Unix File Naming And Pathing

Almost everything in UNIX file system has a name (more precisely, a *path name*). This includes:
- Regular files
- Directories
- Devices (also called **Special Files**)

That said, read and write operations do not behave the same for all files. For example, reading from a directory is not useful and thus is not standardized in most operating systems.

### Directories

Directories contain *references* to other files by pointing the I-Node of other files.

![[Pasted image 20230328222240.png]]

A directory, is a lookup data structure that maps a file name to an I-Node number. So think of it as like a Java `map` object that uses strings as keys and I-Node numbers as values (which is just an integer). All of this mapping is done in the **Virtual File System**, but the actual I-Nodes are in the **Actual File System**.

The map would look something like this:

![[Pasted image 20230328223434.png]]

>[!FAQ] What Really *Is* an I-Node?
>The I-Node implementation in the AFS essentially maps an I-Node number (so an integer) to a *Disk Address*, which contains actual data. As such, I-Nodes differ greatly between operating systems and disk types and thus we don't bother giving any more details about them in this class. Just know that they live in the AFS and can be accessed in the VFS with I-Node numbers.

#### Pathname Resolution

The process of converting a path name to an I-Node number is called **Pathname Resolution**. For example, where does `/home/ag/foo.c` point to?

Well, it starts with `/` which is the root, and thus we need to resolve the string next to it, before the next `/`. As such, we can search for the next `/`, temporarily replace it with `\0`, thus converting it to a C-string, and read from the character after the root identifier `/`.

Doing so, gives us the string `home` and searching for it in the directory map of `root`, gives us I-Node number 18 (using the previous picture). This process of looking up, joggles a lot between the AFS and VFS, and thus is particularly slow. It makes much more sense to let the AFS lookup the entry *all by itself*, which is much faster.

We can now chunk the path for each of it's subdirectories, until we either hit the end of the string and return, or fail for whatever reason.

As you see, we need to go from VFS to AFS and back as many times as there are `/` characters in the path and thus pathname resolution is very slow. As such, there are ways to optimize it.

- Have reference files in one directory that appears in another directory (note that we say a **file**, NOT a **directory**). These are called **Hard Links** and the Syscall `link()` can create them. The reason is that we can create loops if we allow directories to reference directories above them in the hierarchy. This isn't necessarily a problem, but it slows down the implementation and since our hierarchy graph is no longer a DAG, it has no topological ordering, which means that we'll need additional data structures to traverse these hierarchies.
- We also have **Soft Links** or **Symbolic Links**. It is a (special) file that contains the name of another file or directory! The Syscall `symlink` implements this abstraction.

>[!NOTE]
>The shell commands of the Syscalls `link` and `symlink` are aggregated under the command `ln`, where:
>- `ln` by itself invokes `link`
>- `ln -s` invokes `symlink`

So for example:
```bash
ln /unix /etc/image
```
Creates `image` under `/etc` that links to `/unix`:

![[Pasted image 20230328225310.png]]

As you see, this creates a new file under `/etc` named `image` that has the same I-Node number of `/unix` (i.e. 117).

>[!NOTE] A Side Effect of Hard Links
>When we create a hard link from `/etc/image` to `/unix`, which entry is the original?
>What we mean is that:
>- Is `/unix` created under `/etc/image`?
>- Is `/unix` created under the root directory?
>
>We can't tell! This isn't necessarily a problem, and can be resolved with timestamps if needed. We say that "Hard Links are *symmetrical*". Soft Links on the other hand are not symmetrical.

For a Soft Link example:
```bash
ln -s /unix /home/bc/mylink
```
What this does is like the following:

![[Pasted image 20230328230036.png]]

As you see, it just creates a new file under `/home/bc` named `mylink` that points to another file, which will contain the name of the file it is supposed to point to, which is `/unix` here, and specifically labels it as a symbolic link file. 

As such, if try to resolve `/home/bc/mylink`, the VFS will understand that `mylink` isn't a normal file, and thus opens it and reads it content, which is another path, and then resolves to that path instead.

>[!IMPORTANT] Soft Links are NOT Aliases
>An alias is a string substitution, but pathname resolution requires more than that. For one, it requires you to check for permissions at each step. As such, the two are NOT the same. 
>
>More so, Soft Link resolution requires a context switch, while the previous doesn't.

>[!NOTE] Working Directory
>The directory that you are in "right now", is called the working directory. Each process maintains this entry in the kernel, with the following rules:
>
>- Every path not starting with `/` will resolve from the working directory instead.
>- The Syscall `getcwd` returns the current working directory. The shell command `pwd` does the same.
>- The Syscall `chdir` allows us to change it from the user space. The shell command `cd` does the same.

