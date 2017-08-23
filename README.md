# Debugging Python Extensions with GDB

## Why

Hurdles:

- Symbols
- Optimization
- 

## Getting Symbols

- python-dbg package in Ubuntu 

   This is a debug build of python and it's libraries. There are
   additional checks that are done at the interpreter level and any
   extension built against this python will carry the debug flag, have
   optimization turned off, and have it's symbols available. It's the
   easiest way to get a full set of symbols.

- dbgsym package in Ubuntu

## GDB Extensions
## Digging in
## Platform Variations


## Other tools
### LLDB
### valgrind
### dtrace
### Microsoft Unified Debugging
