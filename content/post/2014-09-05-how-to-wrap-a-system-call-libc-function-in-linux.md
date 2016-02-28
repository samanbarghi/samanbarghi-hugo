---
author: Saman Barghi
categories:
- Linux
- Geeky
- Programming
- C
- System call
comments: true
date: '2014-09-05'
title: How to wrap a system call (libc function) in Linux
---

For one of my research projects I had to wrap linux system calls and redirect
them to another thread. In Linux system calls are not invoked directly, but
rather via wrapper functions in
glibc\[[man 2 syscalls](http://man7.org/linux/man-pages/man2/syscalls.2.html)\]. The glibc
wrapper is only copying arguments and unique system call number to the registers
where the kernel expects them, then trapping to kernel mode and setting the
errno if the system call returns an error number \[[man 2 intro](http://man7.org/linux/man-pages/man2/intro.2.html)\].

It is possible to invoke system calls directly by using syscall \[[man 2 syscall](http://man7.org/linux/man-pages/man2/syscall.2.html)\]. But since most programs will rely on glibc functions for system calls, it will be enough to wrap those functions. There are two ways to wrap or override C functions in Linux:

* **Using LD_PRELOAD:** There is a shell environment variable in Linux called
  *LD_PRELOAD*, which can be set to a path of a shared library, and that library
  will be loaded before any other library (including glibc).
* **Using 'ld \-\-wrap=*symbol*':** This can be used to use a wrapper function
  for *symbol*. Any further reference to *symbol* will be resolved to the
  wrapper function. \[[man 1 ld](http://man7.org/linux/man-pages/man1/ld.1.html)\].

I explain each approach later, but first lets write a very simple test file.
I plan to wrap *write* system call and count the total number of characters that
is being written out.

### Test file

Lets write a very simple test file that calls *write* and *printf* to write to standard output:

```
#include <stdio.h>
#include <unistd.h>

int main()
{
    write(0, "Hello, Kernel!\n", 15);
    printf("Hello, World!\n");

    return 0;
}
```
If I run the code I get:

```
$ ./bin/test
Hello, Kernel!
Hello, World!
```

Now I want to see what are the system calls that are being called when running the test file. I use *strace* to see the system calls responsible for writting to the standard output. *strace* is being used to trace system calls and signals. Here is the result:

``` 
execve("./bin/test", ["./bin/test"], [/* 53 vars */]) = 0
brk(0)                                  = 0x2532000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f099cc04000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=128624, ...}) = 0
mmap(NULL, 128624, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f099cbe4000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\320\37\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1845024, ...}) = 0
mmap(NULL, 3953344, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f099c61e000
mprotect(0x7f099c7d9000, 2097152, PROT_NONE) = 0
mmap(0x7f099c9d9000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bb000) = 0x7f099c9d9000
mmap(0x7f099c9df000, 17088, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f099c9df000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f099cbe3000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f099cbe1000
arch_prctl(ARCH_SET_FS, 0x7f099cbe1740) = 0
mprotect(0x7f099c9d9000, 16384, PROT_READ) = 0
mprotect(0x600000, 4096, PROT_READ)     = 0
mprotect(0x7f099cc06000, 4096, PROT_READ) = 0
munmap(0x7f099cbe4000, 128624)          = 0
write(0, "Hello, Kernel!\n", 15Hello, Kernel!)        = 15
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f099cc03000
write(1, "Hello, World!\n", 14Hello, World!)         = 14
exit_group(0)                           = ?
+++ exited with 0 +++
```

As you can see lines 26 and 29 are where the *write* system call related to our
code is being called. Since our goal is to wrap glibc functions, lets check
output of *ltrace* as well. *ltrace* intercepts and records the dynamic library
calls which are called by the executed process \[[man 1 ltrace](http://man7.org/linux/man-pages/man1/ltrace.1.html)\]. Here is the result:

```
__libc_start_main(0x40057d, 1, 0x7fffdd1ec628, 0x4005b0 <unfinished ...>
write(0, "Hello, Kernel!\n", 15Hello, Kernel!
)                                          = 15
puts("Hello, World!"Hello, World!)         = 14
+++ exited (status 0) +++
```
*ltrace* result shows that the *write* function in the code is calling the
*write* function from glibc, but *printf* is calling *puts* from glibc. So we
should be careful here, overriding only the *write* function from glibc will not
cause the *write* system call from *printf* to be wrapped. We need to
differentiate between the final system call and the glibc library call. So in
order to cover both of the cases, I need to override *write* and *puts*
functions. Now lets jump into wrapping these functions.


## Using LD\_PRELOAD

LD\_PRELOAD allows a shared library to be loaded before any other libraries. So
all I need to do is to write a shared library that overrides *write* and *puts*
functions. If we wrap these functions, we need a way to call the real functions
to perform the system call. *dlsym* just do that for us \[[man 3 dlsym](http://man7.org/linux/man-pages/man3/dlsym.3.html)\]:
> The function dlsym() takes a "handle" of a dynamic library returned
       by dlopen() and the null-terminated symbol name, returning the
       address where that symbol is loaded into memory.  If the symbol is
       not found, in the specified library or any of the libraries that were
       automatically loaded by dlopen() when that library was loaded,
       dlsym() returns NULL...

So inside the wrapper function we can use dlsym to get the address of the related symbol in memory and call the glibc function. Another approach can be calling the syscall directly, both approaches will work. Here is the code:

```
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>
#include <string.h>

/* Function pointers to hold the value of the glibc functions */
static  ssize_t (*real_write)(int fd, const void *buf, size_t count) = NULL;
static int (*real_puts)(const char* str) = NULL;

/* wrapping write function call */
ssize_t write(int fd, const void *buf, size_t count)
{

    /* printing out the number of characters */
    printf("write:chars#:%lu\n", count);
    /* reslove the real write function from glibc
     * and pass the arguments.
     */
    real_write = dlsym(RTLD_NEXT, "write");
    real_write(fd, buf, count);

}

int puts(const char* str)
{

    /* printing out the number of characters */
    printf("puts:chars#:%lu\n", strlen(str));
    /* resolve the real puts function from glibc
     * and pass the arguments.
     */
    real_puts = dlsym(RTLD_NEXT, "puts");
    real_puts(str);
}
```

We first declare pointers to hold the value of the glibc functions, we will use these later to get the pointer from *dlsym*. Then we simply implement the glibc functions that we want to wrap, add our code and finally call the real function to perform the intended task.

#### Compiling the shared library

We compile the shared library as follows:

```
gcc -fPIC -shared  -o bin/libpreload.so src/wrap-preload.c -ldl
```

We need to make sure we are generating a position-independent code(PIC) by passing `-fPIC` that is shared `-shared`. We also need to link our library with Dynamically Loaded (DL) libraries `-ldl`, since we are using dlsym in our code.

To run our test code and wrap glibc functions, we simply set `LD_PRELOAD` enviornment variable to the generated shared object file:

```
$ LD_PRELOAD=/home/saman/Programming/wrap-syscall/bin/libpreload.so ./bin/test
write:chars#:15
Hello, Kernel!
puts:chars#:13
Hello, World!
```

`LD_PRELOAD` loads the libpreload.so library before the execution of our code, and thus calling *write* and *puts* will call our wrapper functions inside the library.

## Using *ld \-\-wrap=symbol*

Another way of wrapping functions is by using linker at the link time. GNU linker provides an option to wrap a function for a symbol \[[man 1 ld](http://man7.org/linux/man-pages/man1/ld.1.html)\]:

> Use a wrapper function for symbol.  Any undefined reference to
           symbol will be resolved to "__wrap_symbol".  Any undefined
           reference to "__real_symbol" will be resolved to symbol.

> This can be used to provide a wrapper for a system function.  The
           wrapper function should be called "__wrap_symbol".  If it wishes
           to call the system function, it should call "__real_symbol".

> Here is a trivial example:

>        void *
>        __wrap_malloc (size_t c)
>        {
>         printf ("malloc called with %zu\n", c);
>         return __real_malloc (c);
>        }

> If you link other code with this file using --wrap malloc, then
    all calls to "malloc" will call the function "__wrap_malloc"
    instead.  The call to "__real_malloc" in "__wrap_malloc" will
    call the real "malloc" function.

> You may wish to provide a "__real_malloc" function as well, so
    that links without the --wrap option will succeed.  If you do
    this, you should not put the definition of "__real_malloc" in the
    same file as "__wrap_malloc"; if you do, the assembler may
    resolve the call before the linker has a chance to wrap it to
    "malloc".

Based on the description, we need to implement two function `__real_symbol` and `__wrap_symbol` (in our case `__real_write` and `__wrap_write`), and link the application with our code. Here is the code:

```
#include <stdio.h>
#include <string.h>

/* create pointers for real glibc functions */
ssize_t __real_write(int fd, const void *buf, size_t count);
int __real_puts(const char* str);


/* wrapping write function */


ssize_t __wrap_write (int fd, const void *buf, size_t count)
{
    /* printing out the number of characters */
    printf("write:chars#:%lu\n", count);

    /* call the real glibc function and return the result */
    ssize_t result = __real_write(fd, buf, count);
    return result;
}

/* wrapping puts function */
int __wrap_puts (const char* str)
{
    /* printing out the number of characters */
    printf("puts:chars#:%lu\n", strlen(str));

    /* call the real glibc function and return the result */
    int result = __real_puts(str);
    return result;
}

```

The code is very straight forward, but now lets try to compile the code and link it with our test application.

```
gcc -c src/wrap-link.c -o bin/wrap-link.o
gcc -c src/test.c -o bin/test-link.o
gcc -Wl,-wrap,write -Wl,-wrap=write -Wl,-wrap=puts bin/test-link.o bin/wrap-link.o -o bin/test-link-bin
```

I used *gcc* to pass the option to the linker with `-Wl`, which is equal to calling `ld` with `--wrap` option. Now if I run the code I get:

```
$ ./bin/test-link-bin
write:chars#:15
Hello, Kernel!
puts:chars#:13
Hello, World!
```

## Conclusion
In order to wrap system calls in Linux, one have to wrap related glibc function calls. You have to be careful about the type of system calls you are tryting to override, since various functions might call different functions from glibc, e.g. *printf* calls *puts* from glibc which calls *write* at the end.

There are two ways to do this: 1-Using `LD_PRELOAD` environment variable, 2-using `ld --wrap`. I personally prefer the first approach since if the number of wrapper functions increases I do not have to specify them one by one, as in the second case.

You can find the source code and the related Makefile in the following github repository: [https://github.com/samanbarghi/wrap-syscall](wrap-syscall).