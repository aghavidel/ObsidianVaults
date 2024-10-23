# More On System Calls

As you remember, the Syscall interface is the **Only** valid interface that can be used to interact with the kernel directly. Syscalls are implemented with *traps*, which are actual machine code instructions that you can use to trap into the OS.

Of course, Syscalls may fail to execute, and when that happens in sixth edition Unix, the OS will return an *error code*, which is represented by the global variable `errno` or the **error number**.

So you can do this:
```c
if (write(fd, buffer, bufsize) == -1) {
	// error!
	printf("error %d\n", errno);
	// now use perror
}
```
The number itself is not very helpful, so the function `perror` exists to help us with this. Once `errno` is set (for example, the case where the given file or directory does not exist), you can immediately call `perror` with some arbitrary starting string, like say `perror("Error: ")`, and you would get the string `Error: No such file or directory` printed to the stream `stderr`, which also prints to stout, but can also be used by the OS to *log* the error if the application desires it.

>[!BUG] Common Bug When Using `perror`
>Be careful though, don't mess up the error number before calling `perror`, lest you get ambiguous error messages.

>[!IMPORTANT] Important Note About Address Space Usage
>Not all of the address space for a process is in the hands of a process.
>For sixth edition Unix, of every 4 GB of address space, the last 1 GB belongs to the kernel, so the true address space essentially looks like the following:
>
>![[Pasted image 20230130160218.png]]
>
>What that means, is the the last addressable memory space for the user program is `0xbfffffff` on a 4 GB memory, and starting from `0xc0000000`, the address space belongs to the kernel.
>
>This  might look ugly, but look at how easy it is to trap into the kernel with this!
>
>![[Pasted image 20230130160756.png]]
>
>As the kernel is a singular entity, the same address space of the kernel spans across all other processes. While this may seem counterintuitive, remember that the address space is **just an abstraction!** There really is no such thing as an address space in the physical execution of the CPU.
>The only reason that we separate process address spaces, is that we want to make sure they don't get in the way of each other!

>[!REMINDER] Registers Reminder
>- `EIP` points to the next operation in the `CODE` segment.
>- `ESP` points to the top of the program stack.
>- `EBP` points to the "middle" of the top stack frame (we'll see later where it points exactly)

# Files

The process abstraction that we discussed until now is very isolated. Processes don't bother each other, not because they don't need to, but because they CANT!

So: 
- How would a process communicate something to outside of it's own address space?
- How would a process print something?
- How would a process read something from the keyboard?

This is where our previous notion of *extended address space* comes to mind. Files are **the only way to go outside our address space in sixth edition Unix**.

Files are held in a *directory system*, which is a tree like structure of parents and children file names. The OS may restrict access for a process by making sure that they cannot go further than some amount towards the **root** of the tree (anything that has access to the root, can technically hop to any file that it wants).

>[!IMPORTANT] File Access
>There is some conditions that are required for the process to access a known file:
>
>- The process needs to have access to **all files along the path to reach this file**.
>- The process needs to have the privilege for what it wants to do with the file.
>- The OS should be able to provide a *handle* for the file in question to the process.
>
>So what's a handle? The OS will never give you the main data structure for the file itself, it only gives some interface to work with it (i.e. streams), this is what we call *handles*. For many instances, only one handle may be used at any time, so the OS might have to keep a program waiting or outright deny access to a file, before it can give it to the program. That's why the 3rd condition is necessary.
>
>For this same reason, file handles survive `exec` Syscalls.

## The File Abstraction

- Every file (text, image, executable, etc.) is just a *binary file*, meaning a binary array.
- You may write beyond the limit of the array, but you may NOT read beyond that limit.
- Files are named with a path in the naming tree stored in the kernel.
- System Calls on files do not return, unless the operation is done/runs into an error that kills it. In CS, this is also called *synchronous* execution (sadly, the same name is used for different things). For all of our System Calls in this class, we assume they are synchronous.
  In contrast, an asynchronous Syscall may return before it's operation has finished!

Things that do not obey the third condition exist, and are generally called **Nonblocking IO Syscalls**, and they play a pivotal role in high performance applications, but we won't discuss them in this class. We will however discuss how programs should handle asynchronous events.

To code with this, you would:

```c
int fd;      // This is the handle
char buffer[1024];
int count;

if ((fd = open("/path/to/file", O_RDWR)) == -1) {
	// The file couldn't be opened
	perror("/path/to/file");
	exit(-1);
}
if ((count = read(fd, buffer, 1024)) == -1) {
	// Read failed!
	perror("read");
	exit(-1);
}
```
>[!BUG] Common Bug
>Always read the return code from `read`, it shows you how many bytes the OS put in the buffer. You must not go beyond that many when reading your buffer!

>[!FAQ] Where is the cursor?
>The OS maintains the cursor in an opened handle. So if you start from position 0 and go to position 300 and consume it, the OS will put it on position 301 as long as this handle is kept open.

## Example: The `cat` Program

The `cat` program which you will find in all Unix and Linux editions, reads from a file to the standard output. It's important that we say what these are:

- The `stdin` is a file mapped usually to the keyboard, and in sixth edition Unix has the handler 0.
- The `stdout` is a file mapped usually to the display, and in sixth edition Unix has the handler 1.
- The `stderr` is a file mapped usually to the display, and in sixth edition Unix has the handler 2. It may also have logging functionalities if the user has requested it.

The `cat` program reads from a file in `stdin` and prints out to `stdout`; In case of errors, it prints to `stderr`. You can write it like this (Note this still has some missing pieces, it does not make complete sense at the moment ..):
```c
int main() {
	char buf[BUFFER_SIZE];
	int n;
	const char *note = "Write failed\n";
	
	while((n = read(0, buf, sizeof(buf))) > 0) {
		/*
		 * The write Syscall returns the number of bytes it successfully outputed
		 * on the stream. If for whatever reason, it fails to otuput exactly the given
		 * n bytes on the screen, something is not right!
		 * Thus, you should print an error on this case.
		**/
		if (write(1, buf, n) != n) {
			(void) write(2, note, strlen(note));
			exit(EXIT_FAILURE);
		}
	}
	return EXIT_SUCCESS;
}
```
>[!FAQ] File Pointer vs File Descriptor (Handler)
>- The file descriptor is the same as the handle that we said. It is an integer that identifies the opened file returned by the kernel. You pass this to standard Syscalls like `read` and `write` like above.
>- The File Pointer is a pointer to some memory that contains a standard C struct type `FILE`. It wraps around the file descriptor itself and adds things like the buffer pointer, size and things that make it easier to work with IO. You would have to use `fread` and `fwrite` (standard C library functions) to work with this.

## File Descriptor Allocation

For ***each process***, the kernel maintains a *file descriptor table*, which is an array of pointers to file objects. All file objects represent opened files in the kernel, and the file descriptor returned for them, is essentially just the index into the array of file descriptors.

Whenever a process uses `open` to get a file descriptor, ***the lowest unassociated file descriptor index*** is returned. So if you were to implement this yourself, you should check the file descriptor table, and start from array zero and return the first index that points to `NULL`. So running this:
```c
close(0);
fd = open("file", O_RDONLY);
```
Will **always** assign 0 to `fd` on a UNIX-like system.

Here is another example:

```c
if ((pid = fork()) == 0) {
	close(1);
	if (open("/path/to/output", O_WRONLY) == -1) {
		perror("/path/to/output");
		exit(-1);
	}
	execl("/path/to/primes", "primes", "300", 0);
	exit(1);
}
while(pid != wait(0));
```
>[!IMPORTANT] File Descriptor Table is Persistent Across All Processes
>This means that:
>- Using `exec` family System Calls does NOT change them.
>- Using `fork` also does not change them.
>- The parent and the child will use the exact same descriptor table.

With the above in mind, now, we modify the `primes` program in our previous lecture to output the primes to the file descriptor index 1 (i.e. the standard output). However, the above program actually closes `stdout`, and then opens the output file. Since `stdout` was just closed, opening this file will associate the index 1 with the output file!

Now, we load the `primes` program by using `execl`, which as you remember, we modified it's output to the file index 1! So, we essentially **redirected** the output of this program to the output file!

This is called **IO Redirection**, and is so common that Unix has a syntax for it, what we did above, was equivalent to:
```
user@bash# primes 300 > /path/to/output
```
The same thing can be done with inputs as well, using for example `cat < /path/to/output` would:

- Close `stdin`
- Open `/path/to/output`, which will now have file descriptor index 0
- Call `cat`, and now `cat` will read file index 0 (which is the output file now), and print it on the screen.

>[!FAQ] File Objects
>The file objects pointed to in the file descriptor table, are not just the file descriptor indices. They also contain the **File Context**, i.e. what the process wants to do with the file (this is the read/write or read-write things we saw above).
>
>There is also the cursor position, and how/who opened the file in the first place. Thus, no user program is allowed to actually *modify* file objects, since if they did, they would be able to open a file in read-only, but change to read-write halfway later and start doing nasty things! This is why we use handlers in the first place, to hide these things from users.
>
>See this picture for details:
>
>![[Pasted image 20230130192142.png]]
>
>Here:
>- `refCount`: The number of processes using this file object
>- `accessMode`: read/write access indicators
>- `fileLocation`: The cursor in the file
>- `inodePointer`: A pointer to an `inode` structure. We'll see this later

>[!IMPORTANT] Reference Count
>Each `open` will increment reference count, and each `close` will decrement it. Once this becomes zero, we can safely free this object file in the system-wide descriptor table.

# Multithreaded Programming

Recall, the *thread* is an abstraction of the CPU, and remember that we are not talking about multiple CPUs (i.e. multiprocessing), we are concerned with concurrency, which one of it's implementations is *threading*.

The OS however, needs to provide **concurrency control**, meaning that it should take care to prevent these threads from messing up each other.

## Example: `rlogind`

Here, we'll discuss what we call a *remote login daemon*. 

![[Pasted image 20230130213133.png]]

This program is supposed to open a pseudo terminal on your local machine that is connected to another server that is secured with a username/password.

The daemon essentially maps the input and output of the server, to the input and output of your local machine, using the network between the two systems.

In the beginning, the daemon was created with single threaded codes, which if you go on and actually see it, is pretty ugly and not very easily understood. It also has to use a system call named `select`, which we haven't discussed yet, but let us just say that it is not ideal for this.

With threads however, you can write two pieces of code:

![[Pasted image 20230130213952.png]]

```c
void incoming(int r_in, int l_out) {
	int eof = 0;
	char buf[BUFFER_SIZE];
	int size;

	while(!eof) {
		size = read(r_in, buf, BUFFER_SIZE);
		if (size <= 0) {
			eof = 1;
		}
		if (write(l_out, buf, size) <= 0) {
			eof = 1;
		}
	}
}
```
And:
```c
void outgoing(int l_in, int r_out) {
	int eof = 0;
	char buf[BUFFER_SIZE];
	int size;

	while(!eof) {
		size = read(l_in, buf, BUFFER_SIZE);
		if (size <= 0) {
			eof = 1;
		}
		if (write(r_out, buf, size) <= 0) {
			eof = 1;
		}
	}
}
```
As you can see, these two are effectively the same, simple to understand code, that just read from some stream and put data on the other stream! This is essentially nothing but the `cat` program, used in a different context!

## POSIX Standard

The POSIX standard is a verified standard for writing multi-threaded code. Everything in the world now uses this for the most part (Linux, Mac, Windows, Solaris, etc.).

Systems that implement the POSIX standard, use essentially the same data structure management as a file, so the OS has to maintain a ***Thread Control Block*** and keep track of all new threads. Similar to how we opened files, to open a thread, we also simply use a handle for that thread to avoid having to give the control block to the user program (so instead of a file descriptor, we'll use what we call *thread IDs*).

If you run `man pthread_create` you'll see the following somewhere in the beginning:

```c
#include <pthread.h>

int pthread_create(
	pthread_t *thread,
	const pthread_attr_t *attr,
	void *(*start_routine)(void*),
	void *arg
)
```
Here:
- The pointer `thread`, points to a `pthread_t` type that will contain our thread identifier.
- The argument `attr` **will always be 0 in this class**.
- The third argument is a pointer to a function, which will run as soon as the thread comes alive.
- The fourth argument, is the list of arguments to pass into the start routine.

Here is an example:

```c
void start_servers() {
	pthread_t thread;
	int i;
	for (i = 0; i < 100; i++) {
		pthread_create(
			&thread,   // Thread ID
			0,         // Default Attributes
			server,    // First Procedure
			argument   // Arguments For The First Procedure
		);
	}
}

void *server(void *arg) {
	// Do some service
	return 0;
}
```
In POSIX, all these 100 threads share their address space with the caller thread, and each thread will have to create it's own stack in the space, and set it's first stack frame to be the starting routine, like the following:

![[Pasted image 20230130221316.png]]