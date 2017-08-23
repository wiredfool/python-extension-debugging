# Debugging Python Extensions with GDB

## Why

Hurdles:

- Symbols
- Optimization
- Namespaces

## Getting Symbols

- python-dbg package in Ubuntu 

   This is a debug build of python and it's libraries. There are
   additional checks that are done at the interpreter level and any
   extension built against this python will carry the debug flag, have
   optimization turned off, and have it's symbols available. It's the
   easiest way to get a full set of symbols.

- dbgsym package in Ubuntu

   Ubuntu also has symbols packages available for some releases.  These 
   will get you the symbols for the non-debug builds of the python 
   interpreter, but won't turn off optimization.
   
- Local build

   This is always available to get you a debug build of just your 
   extension. You don't get the same visibility into the python 
   interpreter, but it does give you a package that you can install 
   anywhere you would normally install.

## GDB Extensions

- There are extensions for gdb since version 6 that allow for native views of python objects from within gdb.  It's not necessary, but its quite convenient. 
- Not turned on by default in ubuntu.  Fedora?

## Digging in
## Platform Variations
## pdb vs gdb

## Other tools
### LLDB
### valgrind
### dtrace
### Microsoft Unified Debugging
