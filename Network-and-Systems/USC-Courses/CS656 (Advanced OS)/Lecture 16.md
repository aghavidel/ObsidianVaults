**Paper:** [Shielding Applications From An Untrusted Cloud With Haven](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-baumann.pdf)

# Security In The Cloud

## Typical Trust Model

![[Pasted image 20240415021258.png|500]]

We have seen this OS design:
- User processes never trust each other
- The kernel never trusts any user level process

Here, trust means that care is taken to make sure two processes cannot have direct impact on each others address space.
The sort of things that can happen here are:
- Service denial (i.e. hogging memory or CPU)
- Masquerading as another user 
- Memory corruption (which may not be done on purpose!)
- Snoop on other files or addresses

We have seen OS attempts to resolve these:
- To prevent service denials, kernels can preempt processes or use hardware interrupts to stop hungry processes
- Using permissions and tight definitions of access
- For the last ones, the OS mostly relies on the virtual address mapping to isolate itself from processes and processes from each other. It can't protect *itself* from memory corruption though! We can only hope the kernel developers are careful.

## A Different Trust Model

All of the above solutions, rely on the trust from the users to kernel, but is that really true?
In the cloud, that is absolutely not the case. The kernel lives over the network, in a private data center, whereas the applications, may not even be local to the kernel, and have no reason to assume the kernel is trustworthy?

What can a malicious kernel do?
- Return malicious data on Syscalls, or just incorrect data
- Steal tenant's information and share it with another data center (who gives you your TCP connections?)
- Execute tenant's code with some modification or just incorrectly!

We hope to provide two things:
- **Confidentiality:** Do not leak client data into the kernel
- **Integrity:** Make sure the kernel does not screw the client, it does exactly as the client says (within reason, the kernel should still protect itself)

For now, we do not go beyond these (for example, service denial is not part of this set of constraints).
The only thing that the applications *have* to trust from the cloud, is that the **CPU** is trustworthy. We have seen Mercurial Cores, so those would break this system completely! (which is kind of trivial, but still ..)

## Software Guard Extensions (SGX)

SGX is a technology first provided by Intel CPUs:

![[Pasted image 20240415040112.png|300]]

Using SGX, a user level process (or kernel) can define a series of virtual address regions called an *Enclave*. SGX provides the guarantee that:
- The Enclave is kept in memory until the caller specifically relinquishes it and makes sure only the original owner can access it.
- The data associated with the Enclave in the physical memory is encrypted, and upon request by the owner, it is decrypted on the fly using CPU hardware.

Why does this work for confidentiality and integrity?
- It provides confidentiality, since the real data is encrypted in memory (which means even knowing the physical address by itself won't really help)
- Following from above, it keeps that confidentiality by making sure that when that data is decrypted (i.e. when it is in the registers or CPU cache), then anyone using it must be in the Enclave.
- It provides integrity with respect to the CPU, since as long as the CPU does it's job correctly, then no one other than the Enclave can do anything with this data.

So at the cost of performance and the Enclave resolution overhead, then this does a pretty good job at providing confidentiality and integrity, however:
- Side-channel attacks are probably pretty important to consider now, since the cache can be a weakness
- If the Enclave itself becomes compromised, then there is nothing we can do

This is the main technology that powers Haven, which we will discuss.

### Some Challenges

There are 2 main challenges with using SGX:
- **Backwards Compatibility:** To use SGX, applications should first declare the Enclaves, and of course, older applications don't do that. As such, we need to handle that part.
- **Extended Address Space:** Anything outside of the virtual address map, being files and any file system that lives over the network, and all IO operation associated with them, will still be unsafe.

Haven addresses this by using VMs.

### Background: Virtual Machines

We saw VMs in the context of the OS class, but in brief:

![[Pasted image 20240415041923.png|300]]

- The hardware is managed by a kernel instance called the *Host*, which runs a kernel process known as a Virtual Machine Monitor (VMM). In the cloud, the VMM usually *is* the kernel, as it is the only thing that has hardware access, but that means that the cloud itself does not have a kernel like interface (which is fine for a cloud)
- The VMM can manage multiple user space processes, which run a different kernel. These kernels are called *Guests*. Guests can run their own applications with their own kernel and user level abstraction.
- Any traps into the guest kernel that must use a device, will be routed by the VMM to the host machine and translated between the kernels. 
  There are multiple ways to do this (for example, should the guests modify their kernel to do this, or not?)

The Host-Guest abstraction, where the VMM is replaced by a virtualization layer in the kernel itself, was spearheaded by VMWare, and is directly patented by them (and is the one that normal users would utilize for running VMs on commodity operating systems).

## Haven

Haven has a pretty interesting design. The basic idea is, use a VM to run actual processes, but modify the virtualization layer to use SGX and create Enclaves for each VM.

![[Pasted image 20240415042737.png|400]]

So:
- The app binary is executed within a very light-weight VM
- The hypervisor of the VM is modified, and will then run the binary in its own Enclave

The VM is referred to as a *Pico-process*, and the VM abstraction will protect the host from the guest like with any other VM, however, the pico-process itself should make use of a modified kernel (in this example, a windows 8 kernel) which runs implements a *Drawbridge* interface, which knows how to utilize the SGX drivers, but offloads each call to the host kernel.

This interface, as well as the OS itself (referred to as *Library OS*) is still not trust-worthy, however, its trustworthiness can be *verified*. The library OS and its API will be open-source, and then verified publicly. The minimal design of it should make it easy to verify (or at least, it is the academic argument for this).

The library OS should be defensive, it should verify all up-calls and down-calls to make sure they belong to its Enclave, and it should control its virtual resources.

### Handling Extended Address Space

We still have to deal with files. 
Here is a strawman approach, just encrypt the files. That works right?

No! A malicious kernel can:
- Use a rollback attack to compromise data integrity
- Use a side-channel attack (for example by looking at the meta-data of the file to infer access patterns)

>[!FAQ] Rollback Attacks
>If you are hosting some encrypted data in the kernel, and the kernel is malicious, well it may not be able to figure out what the data is, but it can cause a lot of ruckus!
>For example, the kernel can keep old versions of the encrypted file, and once you modify the real file, it can restore the older ones. Unless the application has a manifest of the original file, it won't figure out something is wrong, since the older data was still a valid encryption and will have a valid decryption!
>
>You may ask, well, SGX has that problem too! And that is technically true, but remember, we are trusting the CPU to do its job correctly.

The solution that Haven used can be seen in the above picture. The File System itself will be private, as part of the Shield Module, which the user can verify, and the disk itself will be encrypted, to protect it from tampering by the Host kernel.

To protect against rollback attacks, the Shield module will implement version control in the file system to keep a manifest about what is happening on the disk.

## Evaluation

The main metrics are:
- The slow-down, the performance of course will take a hit, so we should measure that.
- Breakdown of how SGX affects the performance of individual operations.
- And of course, the harder one, security considerations

![[Pasted image 20240415045223.png|500]]

Here, the paper first addresses the overhead by testing two different applications. 
	a) Is SQL server running TPC-E benchmark
	b) Is Media Wiki on an Apache webserver 

The bars are:
- **Native** is just running directly on the host OS
- **Hyper-V VM** is running directly in normal VMWare workstations
- **Drawbridge** is the lightweight VM that was designed for Haven
- **Haven Host FS** is application running over Haven, but utilizing the file system in the host
- **Haven VHD** is using non-encrypted disk in the host, but the file system is provided by the guest VM as a virtual file system
- **Haven Encrypted VHD** is the just the above, with added disk encryption

The takeaways are:
- Native is fastest, which is expected, it has the least overhead, the degradation seen from native to the VM case is just the virtualization overhead
- The Drawbridge execution in SQL at least is faster, since it is much more lightweight
- Haven over host FS sees a performance drop, since Haven has an extra Library OS layer to handle for SGX
- The VHD is its own file system which is probably not that optimized compared to what is in the host, so there is a performance drop there as well
- Encrypted disks are slow, who could have thought?

However, there is a curious thing to note with the Apache test though. Media Wiki is just a Wiki, which holds plaintext documents for reading and editing. This means that there is a lot of small IO operations, and thus, it means that having to context switch between the guest to the VM to access the file system, becomes extremely slow. This is why Drawbridge and Haven over host FS are so bad.

In contrast, using a virtual FS prevents that, even though the VHD is probably nowhere near as optimized as in the host. Let this be another reminder that context switching is very slow.

For a performance breakdown, there are two parts that we need to consider:
- Memory Allocation overhead, which shows how much SGX itself slows down the process
- Enclave Crossing overhead, which shows how much having this extra layer to cross (as well as flushing the TLB all the time) slows the application down

To this end, the paper *emulates* SGX, by using different delays on each of these operations. You can see the effect here:

![[Pasted image 20240415050809.png|500]]

As you can see:
- Memory Allocation has no impact on SQL server compared to the Apache test, since the Apache web server allocated memory for each request dynamically for scalability, but most database servers use static allocations and increase memory ahead of time since performance is critical. Thus, there is barely any impact for SQL server.
- Enclave crossing heavily impacts both applications. Apache server suffers less since it is very IO bound.

