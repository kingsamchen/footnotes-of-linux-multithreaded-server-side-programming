# A Quick Tour of Compiling, Linking, Loading, and Handling Libraries on Unix

#### Abstract

_In the September/October 2001 issue of _Computing in Science & Engineering_, an I.E.E.E. publication, edited by Paul F. Dubois, three authors, David M. Beazley, Brian D. Ward, and Ian R. Cooke, wrote an article _"The Inside Story on Shared Libraries and Dynamic Loading"_. As we thought that the subject was also interesting for the readers of the CNL, we obtained permission of the I.E.E.E. to make available the original version as a [PDF preprint](http://ojps.aip.org/getpdf/servlet/GetPDFServlet?filetype=pdf&id=CSENFA000003000005000090000001&idtype=cvips), ([Volume 3, Issue 5](http://ojps.aip.org/journal_cgi/dbt?KEY=CSENFA&Volume=3&Issue=5#MINOR5)) [CERN Intranet only], with the following Copyright:_

_

© 2001 IEEE. Reprinted, with permission, from Computing in Science & Engineering--September 2001 Volume 3, Issue 5.

As a complement to that article we present here a few practical examples on how to handle libraries with C++ programs, e.g., NAGLIB, CLHEP, etc., that are used at CERN.

_

Compilers and object files
--------------------------

When building an executable program the compiler converts source files into object files. Each object file contains machine code instructions that correspond to statements in the program source. Generally, object files are broken into a collection of sections corresponding to different parts of the source program. As an example, let us consider the following small C++ program. It is good practive to explicitly specify the namespace std:: when referring to _standard_ classes and methods. This will guarantee maximal compatibility with standard-compliant C++ compilers and run-time libraries.

```c++
#include <iostream>
int main(int, char **) {
  int ny = 2002;
  std::cout << "Merry Xmas, Happy " << ny << "!" << std::endl;
}
```

We can compile it with a C++ compiler (the g++ GNU compiler on Linux, and the native c++ compiler on Solaris, but whatever is explained in the present article should apply to other standard-compliant compilers on different computer platforms as well), and then execute the generated executable program, first on Linux (e.g., lxplus at CERN):

```shell
$ g++ merry.cpp -omerry
$ merry
Merry Xmas, Happy 2002!
```

or similarly on Sun OS 5.7 (e.g., sundev at CERN):

```shell
$ CC  merry.cpp -o merry1
$ merry1
Merry Xmas, Happy 2002!
```

It comes as no surprise that the generated output is identical on both systems. Information about the object file that contains the various compiled sections (machine code instructions, global and local data) and the symbol table describing the source program identifiers can be obtained by the Unix nm, command, which lists symbols from object files. The detailed information provided by the command nm is platform-dependent, and the man pages should be consulted to understand its precise structure. However, we show some salient features below, first on Linux:

```shell
$ nm --print-file-name --demangle merry
merry:080499ac A _DYNAMIC
merry:08049978 A _GLOBAL_OFFSET_TABLE_
     ...
merry:08049a60 B cout
merry:08049928 W data_start
merry:         U endl(ostream &)
merry:08048834 t fini_dummy
merry:08049934 d force_to_data
merry:08049934 d force_to_data
merry:0804883c t frame_dummy
     ...
merry:08048864 T main
merry:08049a8c b object.8
merry:0804992c d p.2
```

and then on SunOS 5.7 (more than 2000 lines).

```shell
$ nm -V -A -R -C merry1
nm: Software Generation Utilities (SGU) Solaris-ELF (4.0)

merry1:

merry1: [Index]   Value      Size    Type  Bind  Other Shndx   Name

merry1: [25]	|         0|       0|SECT |LOCL |0    |24     |merry1:
   ...
merry1: [1274]	|    372276|       0|FUNC |GLOB |0    |UNDEF  |merry1:.div
merry1: [790]	|    372264|       0|FUNC |GLOB |0    |UNDEF  |merry1:.mul
merry1: [582]	|    372072|       0|FUNC |GLOB |0    |UNDEF  |merry1:.stret8
   ...
merry1: [389]	|         0|       0|FILE |LOCL |0    |ABS    |merry1:xsZ9_GTZj8hTFL_5l8EJ
merry1: [374]	|         0|       0|FILE |LOCL |0    |ABS    |merry1:yF0C_cVsuFXDicukF3CL
merry1: [388]	|         0|       0|FILE |LOCL |0    |ABS    |merry1:yJhYas4P93YJaULD27v0
merry1: [162]	|         0|       0|FILE |LOCL |0    |ABS    |merry1:zOtiZS4hQ7CvMDXtQoDv
```

Linkers and linking
-------------------

To build an executable file, the linker (for example, ld) collects object files and libraries. The linker's primary function is to bind symbolic names to memory addresses. To do this, it first scans the object files and concatenates the object file sections to form one large file (text sections of all object files are concatenated, data sections are concatenated, and so on). Then, it makes a second pass on the resulting file to bind symbol names to real memory addresses. To complete the second pass, each object file contains a relocation list, which contains symbol names and offsets within the object file that must be patched. For example, the relocation list for the object file for our simple C++ example looks something like this:

```shell
$ objdump -R merry

merry:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE
080499a8 R_386_GLOB_DAT    __gmon_start__
08049a60 R_386_COPY        cout
08049984 R_386_JUMP_SLOT   endl__FR7ostream
08049988 R_386_JUMP_SLOT   __ls__7ostreamPFR7ostream_R7ostream
0804998c R_386_JUMP_SLOT   __throw
08049990 R_386_JUMP_SLOT   __ls__7ostreamPCc
08049994 R_386_JUMP_SLOT   __deregister_frame_info
08049998 R_386_JUMP_SLOT   __libc_start_main
0804999c R_386_JUMP_SLOT   __ls__7ostreami
080499a0 R_386_JUMP_SLOT   __register_frame_info
080499a4 R_386_JUMP_SLOT   __gmon_start__
```

Similarly, on SunOS 5.7 the dump command with the -L option shows information about dynamic linking and static shared libraries for an objet file.

Static and shared libraries
---------------------------

To improve modularity and reusability one often groups commonly-used functions in _libraries_. In the past, such a library was in the form of an archive (.a file), created like this:

```shell
$ ar cr libmyar.a myobj1.o myobj2.o myobj3.o ...
```

The resulting libmyar.a file is called a _static_ library. Such an archive is merely a collection of raw object files strung together along with a table of contents for fast symbol access (sometimes one even had to manually construct this table of contents with a special utility, such as ranlib.) Static libraries are easy to create and use, but they present a number of software maintenance and resource utilization problems. Hence, nowadays shared or dynamic link libraries are preferred.

When a static library is included during program linking, the linker makes a pass through the library and _adds all the code and data_ corresponding to symbols used in the source program. Thus, when the linker includes a static library in a program, it copies data from the library to the target program. This means that whenever a change to a library is needed, everything linked against that library must be rebuilt for the changes to propagate. Moreover, copying the contents of a library into the target program wastes disk space and memory. For example, if every executable on a system includes its own copy of the C library, the size of these programs increases dramatically, and when active, they waste a considerable amount of system memory because they each store their copies of the library functions.

It is important to note that the linker ignores unreferenced library symbols and aborts with an error when it encounters a redefined symbol.

These days the maintenance and resource problems of static libraries are addressed by using shared libraries or dynamic link libraries (DLLs). The primary difference between static and shared libraries is that using shared libraries delays the actual task of linking to runtime, where it is performed by a special dynamic linker-loader. So, a program and its libraries remain decoupled until the program actually runs. This means that when a bug has to be corrected in a commonly used library the latter can be patched and updated without the need to re-compile and re-link applications referring to the library in question, whose code is only dereferenced when the progam is actually executed. Moreover, optimizations on the system level are possible, e.g., by grouping non self-modifying executable instructions in read-only memory regions that can be shared among processes. Thus, if many programs are running and a large fraction of them include the same library (e.g., the C library), the operating system can load a single shared copy of the library's instructions into physical memory. This significantly reduces memory use and improves system performance.

On most systems, the static linker handles both static and shared libraries. In fact, gcc and g++ are set up to use shared libraries by default. In the example below we first compile and link in static mode, and then in (default) shared mode. In the latter case the location of the shared libraries is determined by the value of the -L option of the g++ command at loading and the setting of the environment variable LD_LIBRARY_PATH at execution, as explained later. Library dependencies are displayed by the ldd (list dynamic dependencies of executable files or shared objects) command.

```shell
# Generate static version of program
# add -v option to g++ command to see how libraries are searched
$ g++ --static merry.cpp   -o merryst
$ ls -l merryst
-rwxrwxr-x   1 goossens dl       1436909 Nov 14 15:51 merryst
$ g++ merry.cpp -lpthread  -o merrysh
$ ls -l merrysh
-rwxrwxr-x   1 goossens dl         13341 Nov 14 16:03 merrysh
$ ldd merrysh
        libpthread.so.0 => /lib/libpthread.so.0 (0x2aacb000)
        libstdc++-libc6.1-1.so.2 => /usr/lib/libstdc++-libc6.1-1.so.2 (0x2aade000)
        libm.so.6 => /lib/libm.so.6 (0x2ab20000)
        libc.so.6 => /lib/libc.so.6 (0x2ab3d000)
        /lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x2aaab000)
```

Similarly, on SunOS 5.7 this becomes:

```shell
# Generate static version of program
$ CC -Bstatic  merry.cpp -o merryst1
$ ls -l  merryst1
-rwxrwxr-x   1 goossens dl        789912 Nov 15 15:55 merryst1
$ CC merry.cpp -lpthread  -o merrysh1
$ ls -l merrysh1
-rwxrwxr-x   1 goossens dl        338240 Nov 15 15:56 merrysh1
$ ldd  merrysh1
        /usr/platform/SUNW,Ultra-60/lib/libc_psr.so.1
        libpthread.so.1 =>       /usr/lib/libpthread.so.1
        libCrun.so.1 =>  /usr/lib/libCrun.so.1
        libm.so.1 =>     /opt/SUNWspro/lib/libm.so.1
        libw.so.1 =>     /usr/lib/libw.so.1
        libc.so.1 =>     /usr/lib/libc.so.1
        libdl.so.1 =>    /usr/lib/libdl.so.1
        libthread.so.1 =>        /usr/lib/libthread.so.1
        /usr/platform/SUNW,Ultra-60/lib/libc_psr.so.1
```

When binding symbols at runtime, the dynamic linker searches libraries in the same order as they were specified on the link line and uses the first definition of the symbol encountered. If more than one library happens to define the same symbol, only the first definition applies. Although symbol binding is normally of not much interest to the final user, sometimers useful information can be obtained by studying the (huge) output generated by setting the LD_DEBUG environment variable to the value bindings before starting your program. For instance, the following command gives almost 1000 lines of output: (we only show the first two and last nine...).

```shell
$ LD_DEBUG=bindings  merrysh
27534:	binding file /lib/libc.so.6 to /lib/libc.so.6: symbol `_nl_current_LC_CTYPE' [GLIBC_2.0]
27534:	binding file /lib/libc.so.6 to /lib/libc.so.6: symbol `_nl_current_LC_COLLATE' [GLIBC_2.0]
27534:	...
27534:	calling fini: /lib/libm.so.6
27534:
27534:	binding file /lib/libm.so.6 to /usr/lib/libstdc++-libc6.1-1.so.2: symbol `__deregister_frame_info' [GLIBC_2.0]
27534:
27534:	calling fini: /lib/libc.so.6
27534:
27534:	binding file /lib/libc.so.6 to /usr/lib/libstdc++-libc6.1-1.so.2: symbol `__deregister_frame_info' [GLIBC_2.0]
27534:	binding file /usr/lib/libstdc++-libc6.1-1.so.2 to /lib/libc.so.6: symbol `_IO_file_setbuf' [GLIBC_2.1]
27534:	binding file /lib/libc.so.6 to /lib/libc.so.6: symbol `_exit' [GLIBC_2.0]
```

Library loading
---------------

When a program linked with shared libraries runs, program execution does not immediately start with that program's first statement. Instead, the operating system loads and executes the dynamic linker (usually called ld.so), which then scans the list of library names embedded in the executable. These library names are never encoded with absolute pathnames. Instead, the list has only simple names such as libpthread.so.0, libm.so.6, and libc.so.6, where the last digit is a library version number. To locate the libraries, the dynamic linker uses a configurable library search path. This path's default value is normally stored in a system configuration file such as /etc/ld.so.conf, which on lxplus at CERN contains the following directories:

```shell
/lib
/usr/lib
/usr/X11R6/lib
/usr/i486-linux-libc5/lib
/usr/local/lib
```

If one wants to extend the set of directories searched one should add the paths to these directories to the LD_LIBRARY_PATH environment variable. It should be mentioned that libraries load in the order in which they were linked, thus for the following command:

```shell
$ g++ mysource.cc -lmy1 -lmy2 -lmy3 ...
```

the dynamic linker loads the libraries in the order libmy1.so, libmy2.so, libmy3.so, etc. If a shared library includes additional library dependencies, those libraries are appended to the end of the library list during loading. For example, if the library libmy2.so depends on an additional library libmy4.so, that library loads after all the other libraries on the link line (in our example after libmy3.so). The load process guarantees that no library is ever loaded more than once with previously loaded copies being used when a library is repeated.

You can follow in detail how the dynamic linker loads libraries by setting the LD_DEBUG environment variable to libs, e.g.,

```shell
$ LD_DEBUG=libs merry
14224:  find library=libstdc++-libc6.1-1.so.2; searching
14224:   search cache=/etc/ld.so.cache
14224:    trying file=/usr/lib/libstdc++-libc6.1-1.so.2
14224:
14224:  find library=libm.so.6; searching
14224:   search cache=/etc/ld.so.cache
14224:    trying file=/lib/libm.so.6
14224:
14224:  find library=libc.so.6; searching
14224:   search cache=/etc/ld.so.cache
14224:    trying file=/lib/libc.so.6
14224:
14224:  calling init: /lib/libc.so.6
14224:
14224:  calling init: /lib/libm.so.6
14224:
14224:  calling init: /usr/lib/libstdc++-libc6.1-1.so.2
14224:
14224:  initialize libc
14224:
14224:  initialize program: merry
14224:
14224:  transferring control: merry
14224:
Merry Xmas, Happy 2002!
14224:
14224:  calling fini: /usr/lib/libstdc++-libc6.1-1.so.2
14224:
14224:  calling fini: /lib/libm.so.6
14224:
14224:  calling fini: /lib/libc.so.6
```

and on SunOS 5.7

```shell
$ LD_DEBUG=libs merry1
28947:
28947: configuration file=/var/ld/ld.config: unable to process file
28947:
28947: find library=libCrun.so.1; searching
28947:  search path=/opt/SUNWspro/lib:/usr/openwin/lib:/usr/dt/lib:/usr/local/lib  (LD_LIBRARY_PATH)
28947:  trying path=/opt/SUNWspro/lib/libCrun.so.1
28947:  trying path=/usr/openwin/lib/libCrun.so.1
28947:  trying path=/usr/dt/lib/libCrun.so.1
28947:  trying path=/usr/local/lib/libCrun.so.1
28947:  search path=/opt/SUNWspro/lib/rw7:/opt/SUNWspro/lib:/opt/SUNWspro/lib:/usr/ccs/lib:/usr/lib  (RPATH from file merry1)
28947:  trying path=/opt/SUNWspro/lib/rw7/libCrun.so.1
28947:  trying path=/opt/SUNWspro/lib/libCrun.so.1
28947:  trying path=/opt/SUNWspro/lib/libCrun.so.1
28947:  trying path=/usr/ccs/lib/libCrun.so.1
28947:  trying path=/usr/lib/libCrun.so.1
28947:
28947: find library=libm.so.1; searching
28947:  search path=/opt/SUNWspro/lib:/usr/openwin/lib:/usr/dt/lib:/usr/local/lib  (LD_LIBRARY_PATH)
28947:  trying path=/opt/SUNWspro/lib/libm.so.1
28947:
28947: find library=libw.so.1; searching
28947:  search path=/opt/SUNWspro/lib:/usr/openwin/lib:/usr/dt/lib:/usr/local/lib  (LD_LIBRARY_PATH)
28947:  trying path=/opt/SUNWspro/lib/libw.so.1
28947:  trying path=/usr/openwin/lib/libw.so.1
28947:  trying path=/usr/dt/lib/libw.so.1
28947:  trying path=/usr/local/lib/libw.so.1
28947:  search path=/opt/SUNWspro/lib/rw7:/opt/SUNWspro/lib:/opt/SUNWspro/lib:/usr/ccs/lib:/usr/lib  (RPATH from file merry1)
28947:  trying path=/opt/SUNWspro/lib/rw7/libw.so.1
28947:  trying path=/opt/SUNWspro/lib/libw.so.1
28947:  trying path=/opt/SUNWspro/lib/libw.so.1
28947:  trying path=/usr/ccs/lib/libw.so.1
28947:  trying path=/usr/lib/libw.so.1
28947:
28947: find library=libc.so.1; searching
28947:  search path=/opt/SUNWspro/lib:/usr/openwin/lib:/usr/dt/lib:/usr/local/lib  (LD_LIBRARY_PATH)
28947:  trying path=/opt/SUNWspro/lib/libc.so.1
28947:  trying path=/usr/openwin/lib/libc.so.1
28947:  trying path=/usr/dt/lib/libc.so.1
28947:  trying path=/usr/local/lib/libc.so.1
28947:  search path=/opt/SUNWspro/lib/rw7:/opt/SUNWspro/lib:/opt/SUNWspro/lib:/usr/ccs/lib:/usr/lib  (RPATH from file merry1)
28947:  trying path=/opt/SUNWspro/lib/rw7/libc.so.1
28947:  trying path=/opt/SUNWspro/lib/libc.so.1
28947:  trying path=/opt/SUNWspro/lib/libc.so.1
28947:  trying path=/usr/ccs/lib/libc.so.1
28947:  trying path=/usr/lib/libc.so.1
28947:
28947: find library=libdl.so.1; searching
28947:  search path=/opt/SUNWspro/lib:/usr/openwin/lib:/usr/dt/lib:/usr/local/lib  (LD_LIBRARY_PATH)
28947:  trying path=/opt/SUNWspro/lib/libdl.so.1
28947:  trying path=/usr/openwin/lib/libdl.so.1
28947:  trying path=/usr/dt/lib/libdl.so.1
28947:  trying path=/usr/local/lib/libdl.so.1
28947:  search path=/usr/lib  (default)
28947:  trying path=/usr/lib/libdl.so.1
28947:
28947: calling .init (from sorted order): /usr/lib/libc.so.1
28947:
28947: calling .init (dynamically triggered): /usr/lib/libCrun.so.1
28947:
28947: calling .init (done): /usr/lib/libCrun.so.1
28947:
28947: calling .init (done): /usr/lib/libc.so.1
28947:
28947: transferring control: merry1
28947:
Merry Xmas, Happy 2002!
28947:
28947: calling .fini: /usr/lib/libCrun.so.1
28947:
28947: calling .fini: /usr/lib/libc.so.1
```

Error messages about missing libraries are usually due to the fact that a shared library is placed in a nonstandard location (not defined in ld.so.conf). In that case you should add the directory containing the supplementary library to the LD_LIBRARY_PATH environment variable. However, in that case you also should have specified the same path with the -L option on the g++ command to verify that all externals are resolved by the libraries at link time.

As libraries are loaded, the system must occasionally perform certain preliminary steps. For example, in C++ programs some objects might have to be created before program startup, and destroyed before a program exists. This situation is handled by the dynamic linker by looking for init() (finit()) functions, that are created by the compiler and contain the necessary code needed to initialize (destroy and clean up) objects and other parts of the runtime environment. Calls to these initialization and finalization are clearly seen in the examples with the LD_DEBUG variable set to libs, shown previously.

Using shared libraries at CERN
------------------------------

Although both archive and shared libraries are offered for most packages, linking with the shared libraries is the default method on Linux with the g++ compiler and linker/loader. However, the shared libraries for HEP applications (e.g. CLHEP, NAGC, Geant4, etc.) are not usually in any of the "standard system directories", so the path to these directories have to be communicated to the compiler/linker (-L option) at compile/link time and the dynamic linker (LD_LIBRARY_PATH environment variable) at run time. Presently, on Linux, there is the additional complication that the default version of gcc (2.91) is not adequate for many C++ applications and the preferred gcc 2.95 has to be specifically requested. For CERN applications there are usually well documented startup scripts (e.g. the "Setting up the user environment" of the Anaphe manual) which provide this functionality automatically.

The three examples that follow are set up to be run on the _standard Linux environment_ at CERN (i.e., they have all been tested on lxplus). On Sun Solaris or ather platforms similar but slightly different command names, setting of environment variables, library paths, etc. might be required, but the overall strategy should remain the same.

### Using CLHEP

An example of using the CLHEP library follows. First we define a few environment variables to set the correct compiler version, and to point to the CLHEP hierarchy (in fact, as shown in the last example, one can get access more simply to the packages by initializing the Anaphe environment scripts).

```shell
# Define specific version of compiler libraries
$ export GCCVER=/usr/local/gcc-alt-2.95.2
# Include compiler in PATH
$ export PATH=$GCCVER/bin:$PATH
# Define which versionof CHLEP we want to use
$ export CLHEPVER=1.7.0.0
# Define where the platform/compiler specific files reside
$ export CLHEPSP=/afs/cern.ch/sw/lhcxx/specific/redhat61/gcc-2.95.2/CLHEP/$CLHEPVER
# Compile the program
$ g++ -I$CLHEPSP/include -o testRandom.o -c testRandom.cc
# Define the path of the shared libraries
$ export LD_LIBRARY_PATH=$GCCVER/lib:$CLHEPSP/lib:$LD_LIBRARY_PATH
# Link the program
$ g++ -L$CLHEPSP/lib -lCLHEP -o testRandom testRandom.o
# Execute the program
$ testRandom
 -0.715106 -0.0593743 0.276891 1.25028 0.126944
```

The C++ program testRandom.cc itself is shown next. It is an example of a random sequence of values according to a gaussian distribution, using a generator based in the Mersenne Twister engine algorithm. This procedure is often used in Geant4. The relative paths of the include files on lines 1 and 2 are dereferenced with the help of the -I option of the g++ command, which specifies the top of the directory contining these include files.

```c++
#include "CLHEP/Random/Randomize.h"
#include "CLHEP/config/iostream.h"

int main(int, char**)
{
   HepInt size=5;
   HepDouble vect[size];

   // Create an instance of the algorithm object to be used
   // as underlying engine for the random generator
   //
   MTwistEngine* aRandomEngine = new MTwistEngine();

   // Instruct the generator to set the engine and use it for
   // for the generator of pseudo-random numbers
   //
   HepRandom::setTheEngine(aRandomEngine);

   // Fill the array 'vect' of random values generated
   // according to a Gaussian distribution centered on zero.
   //
   RandGauss::shootArray(size,vect);

   // Print the values on standard output
   //
   for (unsigned int i=0; i<size; ++i )
     HepStd::cout << " " << vect[i];
   std::cout << std::endl << std::endl;

   // Delete the engine object
   //
   delete aRandomEngine;

   return 0;
}
```

### Running with CLHEP and NAGLIB

We show in this section how to link procedures both from the C library NAGLIB and from the C++ library CLHEP. Consider the following C++ program test.cc:

```c++
/* Test program tst.cc */
#include <iostream>
#include "CLHEP/Vector/ThreeVector.h"

#include <nag.h>
#include <nag_stdlib.h>
#include <nagg05.h>

int main(int, char**)
{
  std::cout << "Clhep test output" << std::endl;
   Hep3Vector v(1.,2.,3.);
   std::cout << v << std::endl;

  Int i;
  Int seed = 0;

  std::cout << "g05cac Example Program Results" << std::endl;
  g05cbc(seed);
  for (i=1; i<=5; i++)
  std::cout << g05cac() << std::endl;
  return 0;
}
```

To compile and link this with gcc 2.95 on lxplus, the location of the compiler and package include files has to be defined, e.g., in tcsh:

```shell
$ setenv G295 /usr/local/gcc-alt-2.95.2
$ setenv PATH $G295/bin:{$PATH}
$ setenv NAG /afs/cern.ch/sw/lhcxx/specific/i386_redhat61/Nag_C/6.0
$ setenv CLDIR /afs/cern.ch/sw/lhcxx/specific/i386_redhat61/CLHEP/new
```

Then the compile and link step producing the default a.out is:

```shell
$ g++ -I$CLDIR/include -I$NAG/include -L$CLDIR/lib -lCLHEP -L$NAG -lnagc -lm -lpthread tst.cc
```

In order to run the application, the directories containing the CLHEP, NAGC and gcc 2.95 libraries have to be given in the LD_LIBRARY_PATH environment variable. The ldd command shows how the libraries are dereferenced.

```shell
setenv LD_LIBRARY_PATH $G295/lib:$CLDIR/lib:$NAG
ldd a.out

libCLHEP.so => /afs/cern.ch/sw/lhcxx/specific/i386_redhat61/CLHEP/new/lib/libCLHEP.so (0x2aac0000)
libnagc.so.6 => /afs/cern.ch/sw/lhcxx/specific/i386_redhat61/Nag_C/6.0/libnagc.so.6 (0x2ab1e000)
libpthread.so.0 => /lib/libpthread.so.0 (0x2aeae000)
libstdc++-libc6.1-2.so.3 => /usr/local/gcc-alt-2.95.2/lib/libstdc++-libc6.1-2.so.3 (0x2aec2000)
libm.so.6 => /lib/libm.so.6 (0x2af09000)
libc.so.6 => /lib/libc.so.6 (0x2af26000)
libstdc++-libc6.1-1.so.2 => /usr/lib/libstdc++-libc6.1-1.so.2 (0x2b01b000)
/lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x2aaab000)
```

Finally, running the program produces:

```shell
./a.out
Clhep test output
(1,2,3)
g05cac Example Program Results
0.795124
0.225717
0.37128
0.225035
0.878745
```

### Predefined package environment variables

Software packages developed at CERN come with setup scripts that define the relevant paths and environment variables needed to use the libraries referenced by the application in question.

As an example, for [Anaphe](http://anaphe.web.cern.ch/anaphe/), both C and Bourne startup scripts are available that users have to source to define all needed environment variables. For instance with the variable LHCXXTOP pointing to the top of the Anaphe hierarchy one could add the following in the user's .profile file (Bourne shell):

```shell
export PATH=/usr/local/gcc-alt-2.95.2/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gcc-alt-2.95.2/lib:$LD_LIBRARY_PATH
export LHCXXTOP=/afs/cern.ch/sw/lhcxx
# Source the startup script. If using versions other than 3.6.3
# just substitute it with the new version.
VER=3.6.3
. $LHCXXTOP/share/LHCXX/$VER/install/sharedstart.sh
```

The VER variable is the version of the software to be used (e.g., 3.6.3 November 2001).

Let us now take part of the Anaphe suite, and see how this works. We choose an example of the fitting and minimization package FML, a C++ library with a rich set of classes. We copied the files from the directory /afs/cern.ch/sw/lhcxx/share/FML/1.2.0.1/fml/examples/stlfit locally. We used the following GNUmakefile (a slight variant of the file provided in the original source distribution):

```shell
include $(LHCXXTOP)/share/FML/1.2.0.1/common/fml-app.mk

# if you want to compile with minuit type: gmake "USE_MINUIT=1"
# by default nagc is used

all:    stlfit

stlfit: stlfit.cpp
        $(C++) $(C_FLAGS) stlfit.cpp -o stlfit  -L$(LHCXX_REL_DIR)/lib $(LIBS_FML) $(LIBS_
VECTOR) -lCLHEP  $(DYNA_LIBS)

clean:
        rm -f stlfit
```

We prepare the executable by running gmake:

```shell
$ gmake "USE_RELEASE=1"

g++ -ansi -pedantic -Wall -Wno-long-long -Wpointer-arith
    -Wnested-externs  -Woverloaded-virtual -Wbad-function-cast
    -Wconversion -fnonnull-objects -pthread -I
     /afs/cern.ch/sw/lhcxx/specific/redhat61/gcc-2.95.2/3.6.3/include
     stlfit.cpp -o stlfit
     -L/afs/cern.ch/sw/lhcxx/specific/redhat61/gcc-2.95.2/3.6.3/lib
     -lFML-nag -lGemini-nag -lnagc -lpthread -lVector -lAIDA_Annotation
     -lHepUtilities -lCLHEP -ldl
```

We see above that that the correct list of libraries, include files, etc. is declared automatically thanks to using the provided GNUmakefile. Finally, we run the example, as follows:

```shell
$ stlfit
default constructor
reading in 1D-vector of size 10 (10, )
------------------------------------
ExtendedFitter setup

Number of points: 9

Startup configuration

       Index        Name       Value       Lower       Upper        Step

           0         amp  -1.598e-01          no          no   1.000e+00
           1        mean  -6.056e-01          no          no   1.000e+00
           2       sigma  -2.169e-01          no          no   1.000e+00

fitting successful!

ExtendedFitter results:
Estimator = 1.057e+01
       Index        Name       Value       Error      Minos+      Minos-

           0         amp  -1.567e-01   2.824e+03          no          no
           1        mean  -6.198e-01   1.338e+04          no          no
           2       sigma  -1.371e-01   7.669e+04          no          no

------------------------------------
ExtendedFitter setup

Number of points: 9

Startup configuration

       Index        Name       Value       Lower       Upper        Step

           0         amp  -2.016e-01          no          no   1.000e+00
           1       slope  -8.835e-02          no          no   1.000e+00

fitting successful!

ExtendedFitter results:
Estimator = 7.842e-12
       Index        Name       Value       Error      Minos+      Minos-

           0         amp   2.220e+00   5.316e-01          no          no
           1       slope  -1.250e+00   4.138e-01          no          no
```

Every non-trivial software development effort should set up a similar working environment, whereby sourcing a script will define the relevant paths and libraries to allow one to run programs in a transparant way.

Constructing shared libraries
-----------------------------

In our various examples we have repeatedly used shared libraries. However we have never shown in detail how to create these. Although this is not too complex in principle for C and C++ source code, the procedure is somewhat involved and system-dependent.

First, when shared libraries and dynamically loadable modules are compiled, often special compiler options have to be used for creating position-independent code (-fpic, -FPIC, -KPIC, and so on). These options tell the compiler to generate code that accesses symbols through a collection of indirection registers and global offset tables. The primary benefit of PIC code is that the text segment of the resulting library (containing machine instructions) can quickly relocate to any memory address without code patches. This improves program startup time and is important for libraries that large numbers of running processes must share, although on the other hand PIC can generate a modest degradation in application performance of a few percent. However, many shared library examples do not need PIC code for their shared libraries and dynamically loadable modules. Hence, if performance is important for a library or dynamically loadable module, you can compile it as non-PIC code. The primary downside to compiling the module as non-PIC is that loading time increases because the dynamic linker must make a large number of code patches when binding symbols.

Another possible issue while constructing a shared library is the inclusion of static libraries (.a files) when building dynamically loadable modules. Indeed, when parts of an application are built as static libraries, the linking process might not do what one expects, since the relevant parts of the static library are simply copied into the resulting shared library object. Thus, when these modules load, all private library copies remain isolated from each other. This can result in grossly erratic program behavior, e.g., with changes to global variables not affecting other modules, functions seeming to operate on different sets of data, and so on. In such a case, static libraries must be converted to shared libraries. Then, when multiple dynamic modules link again to the library, the dynamic linker will guarantee that only one copy of the library loads into memory.

Conclusion
----------

We have explained how to compile, link and load C++ programs (the issues are similar for other compiled languages) and pointed out the advantages of shared with respect to static libraries. In particular, with the help of examples, we have introduced the major options of the g++ command to specify libraries against which a program has to be linked. We also underlined the importance of specifying the correct LD_LIBRARY_PATH environment variable at load time. Finally, we mentioned how software packages, such as Anaphe and Geant4, assist their users by providing them with setup scripts and documentation to allow them work in the correct software environment automatically.

More detailed explanations, as well as a discussion of more advanced features, such as setting up filters, or loading new libraries at run time, are available in the [original publication](http://ojps.aip.org/journal_cgi/dbt?KEY=CSENFA&Volume=CURVOL&Issue=CURISS#MINOR5), which we strongly encourage everybody to read, as it serves as a useful complement to this text. We would like to thank Gabriele Cosmo and Jakub Moscicki for their help while preparing the examples, and Andreas Pfeiffer for several suggestions that improved the accuracy of the text.

---
1. 见书 P424
2. 原文链接：https://ref.web.cern.ch/ref/CERN/CNL/2001/003/shared-lib/Pr/
