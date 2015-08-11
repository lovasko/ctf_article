# CTF Article
Daniel Lovasko

In its entirety, type information can be used to understand the mass of bytes
that have no other metadata attached. By knowing the base type attributes such
as the name, encoding and the byte size of an integer, we are able to correctly
assign name to a piece of memory and render the bytes appropriately - as a signed
or an unsigned value. Understanding more complex types such as unions or
structs, opens the possiblity to compose the base types, identify the offsets of members
and recursively dive into each member type. Such traversal and further
interpretation is transformable into a useful tool for programmers and
sysadmins alike. This article will focus on one of the type information
formats, the Compact C Type Format - its description, current and potential
usage.

## What is CTF?
The Compact C Type Format (CTF) is a debugging information format targeted on
the type system of the C99 language. It was conceived at Sun Microsystems while
working on the Modular Debugger and adopted by other significant tools onwards.
One central design goal of the CTF format is contained in its name:
compactness. The format utilises all kinds of techniques, such as bit fields,
ZIP compression or deduplication, to achieve the most minimal form.

## How does CTF play with FreeBSD?
Every FreeBSD installation (which archs?) since the version (version?) already
ships with a CTF data set generated for the GENERIC kernel and its modules. Two
major CTF consumers are the DTrace software suite and the Modular Debugger, out
of which only DTrace was ported to FreeBSD. Further users of the data set are
the CTF tools such ctfdump, ctfmerge, ctfquery and few others.

## How does CTF compare with DWARF?
As far as the type information correctness is concerned, they amount to be an
equal opponents. It is important to note though that DWARF encompasses much
more than just the type information, providing mapping between program counter
and source code lines, macro information and address ranges among others. But
the area where CTF shines is the on-disk size. The following comparison was
performed on a FreeBSD 10.0-RELEASE with the GENERIC kernel. Being fair, only
the `.debug_info` and `.debug_str` ELF sections are being counted into the
DWARF overall sum.

### Graph CTF vs DWARF on-disk size

### What does this mean?
The observable drop in on-disk space requirements allow CTF to be a debugging
format of choice in situations where this metric is of a critical importance -
examples of such might be all production systems or embedded devices.

### How was this computed?
In order to encapsulate the logic of size measurements, we created a utility
`ctfmemusage` [2] that offers size comparisons between the on-disk size of CTF
data set and the DWARF data set.
Moreover, it is possible to measure the ratio between the on-disk and in-memory
representations of the CTF implementation. As of now, the in-memory inflation
is approximately XYZ with with variance of XYZ on a set of XYZ samples.

## How does CTF behave on different architectures?
The best way to describe its behaviour is to say that CTF always follows the
truth. If your `size_t` is 32 bits wide, CTF would agree with that. The same
applies in a situation when your `size_t` is 64, 37 or 2 bits wide. This is a
very powerful feature that allows CTF to be a unifying layer on top of
different platforms, always serving the correct type metadata. 

### Read the `clock_res` of a kernel dump
FreeBSD stores its internal clock resolution in a kernel global variable
`clock_res`. An implementation of the CTF format, `libctf` [3], along with the
Kernel Virtual Memory library `libkvm` allow us to write a small utility that
reads the value of the `clock_res` symbol of any core dump.

#### Pseudocode
```
kernel_ctf = ctf_file_read("/home/user/dumps/kernel1");
clock_res = find_symbol(kernel_ctf, "clock_res");
kvm_read(kvm_handle, clock_res.address, &result, clock_res.size);
```

#### Notes on pseudocode
A real implementation of such program is indeed possible and does not differ
from the pseudocode significantly. A full source can be found at [1].

Of course, the specific variable, in our case the `clock_res`, was chosen
fairly randomly. The same code can be used to inspect any other kernel
variable. An extension of this idea can be found at the end of the article.

## What are the limitations of CTF?
With great power comes great responsibility and with great space optimisations
come great limitations. And CTF is no exception. To achieve its compactness,
CTF makes certain assumptions about the domain of each metric. For example,
each type gets assigned a numeric ID to identify it uniquely. The on-disk
format uses 16 bits to denote the ID, therefore posing a limitation of maximum
65536 types. A compilation unit with more types would not be able to store them
all. Such tight limit is posed on so a called _kind_, essentially a type of a
type - integer, floating point number, pointer, typedef, etc. Only four bits
were allocated for this purpose - the current second version of the CTF format
already utilises 13 out of 16 possible values.

Naturally, more of such limitations exist but the examination of all would
require much more space than this article has.

### Does that affect FreeBSD in any way?
Linear regression is a simple, yet powerful-enough method on how to
predict future behaviour. In our case, we will track the past values of the
mentioned metrics and predict the moment in time when they will _hit the roof_
and therefore pose a problem in the format usability. 

#### Type ID limitation within FreeBSD kernel versions


#### Type kind ID limitation within C standards
C89 - does not have restrict, C11 needs new _Atomic qualifier

### How did you get these numbers? (toolset exploration)

### Is there a solution to this situation?
Yes and no. The current second version of the CTF format is completely rigid
and does not allow any changes to the domain sizes. In case that the next
iteration of the format would adopt the self-describing principle, all
domain-related limitations would be solved. 

## What else can we do with CTF?
### Kernel debugger pretty-printing
Memory correctness plays the key role in any algorithm. Being able to swiftly
and efficiently examine arbitrarily complex data structures is a mandatory
trait of every modern toolchain. The GNU Debugger 'gdb', Modular Debugger
'mdb' and the LLVM Debugger 'lldb' fulfil the expectations with respect
to userland processes. Unfortunately, the kernel debugger DDB on FreeBSD (and
other systems using it) has been shipped without such functionality from
its inception many decades ago. Since the CTF data set is available for the
kernel image and its modules out of the box, it can be used to provide in-depth
view of the data structures that are being used by the kernel.

#### Status quo
Currently, DDB offers two techniques to access the memory which are affected by
many limitations, either lack of modularity or excessive straightforwardness.

##### Low-level examination
The 'examine' command in conjecture with its options such as
'x' or 'f' is used to examine optional number of memory blocks. The
fact that the user must specify the encoding of the memory is a .

This feature of DDB is undoubtedly important in scenarios when a specific
bit or byte needs to be analysed, but falls short in examining complex
structures, as it literally forces user to use pen and paper to map the output
values of the data structure.

##### Hard-coded structures
The 'show' command improves upon the user-facing simplicity of th 'examine'
command by providing support for pretty-printing a restricted set of
data structures. Such structures are e.g. 'struct bio' or 'struct mutex'.

The problem with this approach lies in its _code change linearity_: every
time a data structures member changes, the corresponding code in DDB needs to
adapt. The same applies in situations when a new data structure is introduced -
a new pretty-printing code needs to be written.

#### CTF-enabled approach
The logical extension of the 'show' command is to provide a way of encoding
type information during the compilation process and being able to read such
data later during the debugger runtime - a task that CTF is a great fit for -
so that there will be no need to alter the DDB source code in case of a
structure change in the kernel code-base.

### Architecture-agnostic kernel virtual memory access
FreeBSD features many popular tools that enable the user to study the behaviour
and properties of the live system kernel just as on-disk kernel core dumps.
Areas of interest cover processes and threads (procstat), network stack
(netstat) or I/O and filesystems (iostat, nfsstat). One major missing feature
of these tools is that the kernel core dump must originate from a compatible
architecture, which usually means the exact same one. Just as described above
in the example with the kernel dump variable `clock_res`, CTF can be used to
provide the abstraction layer above the architecture differences, and the stat
tools can gain leverage by applying this method. 

[1] TBD
[2] https://github.com/lovasko/ctfmemusage
[3] https://github.com/lovasko/libctf

