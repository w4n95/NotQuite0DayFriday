# Multiple Bugs in NITRO NITF Parsing Library

### Overview

GRIMM fuzzed the latest release (at the time of the writing of this report) of 
the NITRO NITF parsing library and identified
several memory corruption issues. These issues impact applications which use
NITRO to parse and modify NITF files from untrusted sources. Exploitation of
these issues is likely dependent on the specific application, but may allow an
attacker to obtain remote code execution or cause a denial of service. The two
bugs which may allow an attacker to obtain code execution (bugs 1 and 2 below)
have been fixed in the master branch of NITRO, but the fixes not been included
in a NITRO release.

### Exercising
Download and compile NITRO 
[version 2.9.0](https://github.com/mdaus/nitro/releases/tag/NITRO-2.9.0). Then,
the crashes can be seen in the bundled `show_nitf` application.

```
# Download and build NITRO 2.9, via the instructions in the NITRO README.md
# https://github.com/mdaus/nitro#sample-build-scenario-1
# Assumes the NotQuite0DayFriday files are in /tmp/demo
$ wget 'https://github.com/mdaus/nitro/archive/NITRO-2.9.0.tar.gz'
$ tar xf NITRO-2.9.0.tar.gz
$ cd nitro-NITRO-2.9.0/
$ cp /tmp/demo/poc.c modules/c/nitf/apps/
$ python waf configure --enable-debugging --prefix=installed
$ python waf build
$ python waf install
$ cd installed/bin/

# Run the example show_nitf utility to trigger the bugs
$ ulimit -c unlimited
$ ./show_nitf /tmp/demo/bugs/bug1_rip.ntf
Segmentation fault (core dumped)
$ gdb -q show_nitf core
Reading symbols from show_nitf...done.
[New LWP 41046]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Core was generated by `./show_nitf /tmp/demo/bugs/bug1_rip.ntf'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000562e67b3f450 in nitf_TRE_destruct (tre=0x7ffe491b4ff0) at ../modules/c/nitf/source/TRE.c:104
104                 (*tre)->handler->destruct(*tre);
(gdb) x/i $rip
=> 0x562e67b3f450 <nitf_TRE_destruct+82>:       call   rax
(gdb) info registers rax
rax            0x47464544434241 20061986658402881
(gdb) quit
$ ./show_nitf /tmp/demo/bugs/bug1_crash.ntf
Segmentation fault (core dumped)
$ ./poc /tmp/demo/bugs/bug2_crash.ntf > /dev/null
Segmentation fault (core dumped)
$ ./show_nitf /tmp/demo/bugs/bug3_crash.ntf
Segmentation fault (core dumped)
$ ./show_nitf /tmp/demo/bugs/bug4_crash.ntf
Segmentation fault (core dumped) (although the OOM killer may kill the process 
before the segmentation fault occurs)
$ ./show_nitf /tmp/demo/bugs/bug5_crash.ntf
Segmentation fault (core dumped)
$ ./poc /tmp/demo/bugs/bug6_hangs.ntf > /dev/null
# Hangs forever
```

Exploitation of bug 1 may be dependent on specific library versions and stack
layout. It was tested on Ubuntu 18.04; other versions will still crash, but may
not obtain control of RIP.

Bugs 2 and 6 are within the `nitf_Record_unmergeTREs` function, which is only
called while writing NITF files. As such, there is not a bundled application to
trigger them. Instead, the poc.c test program has been created to illustrate the
bugs. It's a modified version of `show_nitf` which calls the
`nitf_Writer_prepare` function to trigger the bugs.

### Details

GRIMM engineers implemented an [AFL](https://github.com/AFLplusplus/AFLplusplus/)
fuzzing driver for NITRO. This fuzzer was then run against NITRO version 
2.9.0, on GRIMM's servers using a large number of CPUs.
Afterwards, GRIMM binned the resulting crashes and determined the root cause of
each of the bugs. This analysis investigated the use case of opening an attacker
controlled NITF file with NITRO. The following bugs may affect NITRO's
confidentiality, integrity, or availability:

1. In the `readTRE` function in `modules/c/nitf/Reader.c`, the `tre` pointer on
the stack is not initialized upon creation. Then later, if the function hits an
error condition before it is assigned a value, the `tre` pointer will be used
without being initialized. Specifically, it will be passed to the
`nitf_TRE_destruct` function which will dereference it and then call a function
pointer from within the dereferenced value. As such, if an attacker can control
the previous stack contents, by using the NITF file contents to direct the
execution flow, they may be able to exploit this vulnerability. Depending on the
exact use case and deployment of NITRO, exploit mitigations, such as ASLR, may
prevent exploitation.
    
    This vulnerability has since been [fixed in the master branch of
    NITRO](https://github.com/mdaus/nitro/commit/40d66294ab2a7768edd31644d1d1e3004e923fd5)
    by instantiating the `tre` variable to `NULL` on function entry. However, it
    was not marked as a vulnerability, and as such any products using NITRO may
    not incorporate the fix into their product.

This vulnerability was assigned CVE number CVE-2021-26498.

2. In the `nitf_Record_unmergeTREs` function in `modules/c/nitf/Record.c`, the
`overflow` pointer on the stack is not initialized upon creation. Then later,
within the `UNMERGE_SEGMENT` macro, the `overflow` pointer is dereferenced in a 
call to the `moveTREs` function as the destination extension list to which a TRE 
struct is appended. While this vulnerability has a less obvious path to 
exploitation than the `readTRE` vulnerability, given the right use case and 
stack alignment, it may also be exploited.
    
    This vulnerability was fixed in NITRO in the same commit as the previous
    `readTRE` vulnerability. However, while the fix initializes the `overflow`
    variable to `NULL`, it does not add code to check for `NULL` before
    dereferencing `overflow`. As such, this bug can still cause a crash by
    triggering a `NULL` pointer dereference.
    
    The remaining `NULL` pointer dereference vulnerability can be mitigated by
    modifying the `UNMERGE_SEGMENT` macro to check the `overflow` variable for
    `NULL` and erroring out accordingly.

This vulnerability was assigned CVE number CVE-2021-26497.

3. The [`defaultRead`
function](https://github.com/mdaus/nitro/blob/5c8c30b1c95/modules/c/nitf/source/DefaultTRE.c#L88)
in `modules/c/nitf/source/DefaultTRE.c` does not properly check for overflows
within the `length` parameter, which is an unsigned 32-bit integer. This parameter is
taken directly from the NITF file without being checked. If the read length is
`UINT32_MAX`, i.e. `2**32 - 1`, the `NITF_MALLOC(length + 1)` call will overflow
and `malloc` will allocate a 0-byte buffer. Later, in the
`nitf_TREUtils_readField` function, the 0-byte buffer is used in a call to
`memset` with the original length. Then, the NITF TRE contents are copied to
the buffer via `nitf_IOInterface_read`. As the `memset` call uses the original
length, it will crash once the code writes beyond the end of the heap.
    
    This bug is likely unexploitable beyond a denial of service. While the bug
    would cause a standard heap overflow with attacker controlled content (via
    the `nitf_IOInterface_read` function), the crashing `memset` beforehand
    prevents the code from reaching this path. As such, an attacker may be able
    to use this bug to affect NITRO's availability. However, it's unlikely that 
    an attacker could utilize this bug to obtain code execution.
    
    This vulnerability can be mitigated by modifying the `defaultRead` function
    to check the `length` parameter for an integer overflow and exiting with an
    error if detected.

This vulnerability was assigned CVE number CVE-2021-26501.

4. The [`readBandInfo`
function](https://github.com/mdaus/nitro/blob/5c8c30b1c95/modules/c/nitf/source/NitfReader.c#L1412)
in `modules/c/nitf/source/Reader.c` (since renamed to NitfReader.c) does not
properly check for overflows in the calculation of the allocation for the band
information. This function reads the number of lookup tables and bands from the
NITF file, multiplies them, and allocates the result. However, if the two
unsigned 32-bit integers multiply to a value larger than `UINT32_MAX`, i.e.
`2**32 - 1`, then the value will overflow and the function will allocate an
incorrectly sized buffer. This buffer is then passed to the `readField`
function, which initializes it via `memset`. Then, the NITF band info is
copied to the buffer via `nitf_IOInterface_read`. As the `memset` call uses the
original length, it will crash once the code writes beyond the end of the heap.
    
    This bug is also likely unexploitable beyond a denial of service. Similar to
    the previous bug, this bug would cause a standard heap overflow with
    attacker controlled content (via the `nitf_IOInterface_read` function), but
    the crashing `memset` beforehand prevents the code from reaching this path.
    As such, an attacker may be able to use this bug to affect NITRO's
    availability, but it's unlikely they could utilize this bug to obtain code
    execution.

    This vulnerability can be mitigated by modifying the `readBandInfo` function
    to validate that the product of the `numLuts` and `bandEntriesPerLut`
    variables does not overflow and exiting with an error if it does.

This vulnerability was assigned CVE number CVE-2021-26499.

5. The [`readRESubheader`
function](https://github.com/mdaus/nitro/blob/5c8c30b1c95/modules/c/nitf/source/NitfReader.c#L970)
in `modules/c/nitf/Reader.c` (since renamed to NitfReader.c) does not allocate
the `subhdr` variable's `subheaderFields` field before trying to read into it
via the call to `readField`. As such, the `subheaderFields` field is `NULL` and
ends up causing a crash via a `NULL` pointer dereference in `readField` during a
`memset` call. Depending on how NITRO is deployed, this bug may allow an
attacker to cause a denial of service. However, as the attacker cannot control
the value of the `subheaderFields` field, it's unlikely they could utilize this
bug to obtain code execution.
    
    This vulnerability can be mitigated by allocating the `subheaderFields`
    field in the `subhdr` variable, via the `nitf_TRE_createSkeleton` function,
    similar to the `readDESubheader` function.

This vulnerability was assigned CVE number CVE-2021-26500.

6. The [`nitf_Record_unmergeTREs`
function](https://github.com/mdaus/nitro/blob/5c8c30b1c95/modules/c/nitf/source/Record.c#L2182)
iterates over each of the scanned NITF's labels, unmerging the extension
sections. However, the `while` loop used to iterate over the labels does not
increment the list pointer (typically done via a call to
`nitf_ListIterator_increment`). Thus, this loop will never terminate and NITRO
will hang forever. Depending on how NITRO is deployed, an attacker may able to
use this bug to cause a denial of service.
    
    This vulnerability can be mitigated by incrementing the `segIter` iterator,
    via a call to `nitf_ListIterator_increment`.

This vulnerability was assigned CVE number CVE-2021-26502.

7. The [`defaultRead`
function](https://github.com/mdaus/nitro/blob/5c8c30b1c95/modules/c/nitf/source/DefaultTRE.c#L88)
in `modules/c/nitf/source/DefaultTRE.c` contains a memory leak. At the beginning
of the function, a buffer with a user controlled length is allocated to the
`data` variable. However, if an error occurs, execution will jump
to the `CATCH_ERROR` label, which does not free that buffer. In deployments
where NITRO is used in a long-lived process that scans multiple NITF
files, this bug may allow an attacker to cause a denial of service.
    
    This vulnerability can be mitigated by conditionally freeing the `data`
    variable in the error handler, similar to the `descr` variable.

This vulnerability was assigned CVE number CVE-2021-26503.

### Timeline
* 2021.01.26 Notified Nitro maintainer & US CERT; applied for CVE
* 2021.01.26 Maintainer acknowledged our report
* 2021.01.27 Maintainer released 2.10.0, which fixed the reported issues
* 2021.01.29 Public disclosure
* 2021.05.12 Received CVEs from MITRE