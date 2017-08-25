# Debugging Python Extensions with GDB

## Why

Python is a nice, safe language that has ways of adding extensions
that behave badly. So sometimes we need to dig into the extensions as
it's running and see what's really happening under the hood.

This is aimed at people who have a decent understanding of python, and
a little bit of an understanding of C and how to use it.

Hurdles:

- Python land vs. C land.
- Symbols
- Optimization
- Namespaces


## Getting Symbols

- python[3]-dbg package in Ubuntu 

   This is a debug build of python and it's libraries. There are
   additional checks that are done at the interpreter level and any
   extension built against this python will carry the debug flag, have
   optimization turned off, and have it's symbols available. It's the
   easiest way to get a full set of symbols.
   
   Once this is installed, there will be additional python binaries in
   `/usr/bin/python-dbg` or `/usr/bin/python3-dbg`

- dbgsym package in Ubuntu

   Ubuntu also has symbols packages available for some releases.  These 
   will get you the symbols for the non-debug builds of the python 
   interpreter, but won't turn off optimization. There are warnings
   that they don't interoperate well with the `-dbg` packages, so
   choose one. 
   
- Local build

   This is always available to get you a debug build of just your 
   extension. You don't get the same visibility into the python 
   interpreter, but it does give you a package that you can install 
   anywhere you would normally install.

   `CFLAGS='-g -O0' python setup.py build_ext install`

   (or, in my favorite project, `make debug`. Makefiles are good.)

## GDB Extensions

There are extensions for gdb since version 6 that allow for native
views of python objects from within gdb.  It's not necessary, but its
quite convenient.

Not turned on by default in ubuntu.  Fedora?

### GDB 6 vs 7

`Tools/gdb/libpython.py` from the build directory. Not included in the
`-dbg` or main python packages on Ubuntu. This is for gdb 7. This is
also left in the base build directory of Python. 

Enabling in `~/.gdbinit`: 
```
add-auto-load-safe-path /path/to/checkout
```


`/usr/share/doc/python[version]/gdbinit.gz`  Despite the documentation
in the package, these are in the versioned doc directory, not the
python directories. This is enabled by copying into your `.gdbinit`



## Digging in

### Small Test Case
### Segfault
### Not so small test case

## Platform Variations
## pdb vs gdb

## Other tools
### LLDB
### valgrind
### dtrace
### Microsoft Unified Debugging

## References

- https://wiki.ubuntu.com/Debug%20Symbol%20Packages
- GDB Support https://docs.python.org/devguide/gdb.html 


temp tab dump:
- https://stackoverflow.com/questions/7412708/debugging-stepping-through-python-script-using-gdb
- http://bugs.python.org/issue8032
- https://stackoverflow.com/questions/14885328/what-is-needed-to-use-gdb-7s-support-for-debugging-python-programs
- https://github.com/python/cpython/pull/3153
- https://docs.python.org/devguide/gdb.html
- https://stackoverflow.com/questions/41160447/cant-enable-py-bt-for-gdb
- https://sumitkgaur.wordpress.com/2014/05/13/python-debugging/
