---
title: On the art of resource conservation in software systems
---

Something I recently pondered on was how the software industry has consistently been developing novel mechanisms for efficiently sharing and reusing resources.
Memory is probably the most straightforward resource to reuse, but other resources need to be shared as well.
Below, I listed some examples that come to my mind -- you'll see they range from simple data structures to complex systems.

- [Multithreading](https://en.wikipedia.org/wiki/Multithreading_(computer_architecture)) is a mechanism that allows multiple executions of a program fragment to run concurrently while being able to reuse the same executable code and values of dynamically allocated variables and non-thread-local global variables in memory.
- [Dynamic linking](https://en.wikipedia.org/wiki/Dynamic_linker) is a mechanism that allows different program binaries to reuse the same low-level libraries (like libc) without requiring shared libraries to be loaded multiple times in memory or duplicated within each binary.
- [Containerization](https://en.wikipedia.org/wiki/Containerization_(computing)) is a mechanism that allows applications and their dependencies to be packaged together and run uniformly across many environments as if on its own operating system, without needing an entire operating system to be packaged and transcluded into every container. (For comparison, see virtual machines).
- [Kernel same-page merging](https://en.wikipedia.org/wiki/Kernel_same-page_merging) is a mechanism that makes it possible for a hypervisor system to share memory pages that have identical contents between multiple processes or virtualized guests.
- [Browser caching and CDNs](https://en.wikipedia.org/wiki/Web_cache) are mechanisms that allow files to be actively stored for reuse, so even when files are requested by different websites or different users, they only need to be computed or retrieved from the original server once.
- [Kubernetes](https://en.wikipedia.org/wiki/Kubernetes) is a container orchestration mechanism that allows large numbers and varieties of containers to be run concurrently on a networked group of computers (aka "cluster"), scheduling the work dynamically so that it's not necessary for workloads to be fixed to individual computers, or individual computers to be restricted to particular workloads. Put another way, Kubernetes lets you treat a group of computers as a pool of compute.
- [Multitenancy](https://en.wikipedia.org/wiki/Multitenancy) is a pattern where multiple users or customers ("tenants") will be served using a shared instance of software (for example, reusing the same database, or GitLab instance). This allows the fixed-cost resources needed or computations performed by the software to be amortized among all N tenants.
- [Interning](https://en.wikipedia.org/wiki/Interning_(computer_science)) is a mechanism that allows many popular programming languages to avoid reallocating memory for immutable data that is repeated in many parts of a program, such as strings.
- [Ropes](https://en.wikipedia.org/wiki/Rope_(data_structure)) and [tries](https://en.wikipedia.org/wiki/Trie) are data structures that allow collections of strings to be stored without requiring common parts of the string(s) (like repeated prefixes or substrings) to be stored multiple times.

Something interesting is that in many examples, like dynamic linking or string interning, we're able to improve system efficiency by *reducing* the number of copies of data in a system, while in other examples, like caching, we're able to improve efficiency by *increasing* the number of copies of data in a system.
Kind of weird, huh?

There are many systems and standards that are equally if not more useful than the ones above, but I wasn't able to find a clear narrative framing for them in terms of resource reuse or resource sharing.
For example, [HTTP](https://en.wikipedia.org/wiki/HTTP), [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol), and [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) are fundamental protocols, but they mainly solve communication problems.
[AWS S3](https://en.wikipedia.org/wiki/Amazon_S3) and [Postgres](https://en.wikipedia.org/wiki/PostgreSQL) are useful tools for storing data, but they mainly solve issues of atomicity and durability, not those of resource use.

Nonetheless, I think it's fun to try and see where commonalities can be found between them. Fittingly, I've noticed that the ideas used in one place can often be *reused* in many other places.

When I think about resource reuse, I sometimes think about all the systems I've worked on.
Are there places where it feels like there's still a lot of duplication happening?
Is it because of hardware limitations?
Is it because we don't have the right interfaces or boundaries in place?
Or is it because it's hard to share certain kinds of resources without making the design insecure?

If there's anything we can learn from history, it's that there are many mechanisms for resource sharing that have yet to be designed.
I don't believe I can predict the future (and I certainly don't believe others who claim to!) so if I had to make guesses at what these mechanisms are, a lot of them would probably be wrong.
But I think it's more interesting to guess and be willing to be wrong than to simply sit and grumble about how things are.

Here are some of the ideas I had.
Some of them might not be practical.
That said, if any of these already exist, please let me know!

- ??? is a mechanism that allows inference to be performed on a variety of large language models while sharing a large subset of weights between the models so that more models can fit in-memory and fewer GPUs are required to run similar models.
- ??? is a mechanism that allows code from programming languages with disparate calling conventions, ABIs, or object models to be translated between each other with minimal overhead (imagine by translating a Python function and program call stack into a Lua function and call stack, or vice versa), enabling cross-language interoperability and code reuse.
- ??? is a mechanism that allows garbage collection to be gradually added to GC-free programs without introducing [function-coloring](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) problems or penalties for applications that choose not to use garbage collection (imagine like gradual typing but for memory management).
