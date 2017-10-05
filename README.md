# Debugging Python Extensions with GDB

## Why

Python is a nice, safe language which can run extensions that behave
badly, causing interpreter crashes, memory corruption, or memory
leaks. Sometimes we need to dig into an extension as it's running
and see what's really happening under the hood. This talk will cover
an introduction to working with Python C-API extensions in gdb: setting up
your gdb environment, common C-level bugs that can be found, and some
of the challenges of using gdb in a dockerized system.

The talk is aimed at people who have a decent understanding of python,
and a little bit of an understanding of C and how to use it. By the
end of the talk, attendees should be able to isolate a crashing bug in
an extension.

Hurdles:

- Python land vs. C land.
- Symbols
- Optimization
- Namespaces


## Getting Symbols

- python[2|3]-dbg package in Ubuntu 

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
   choose one. https://wiki.ubuntu.com/Debug%20Symbol%20Packages
   
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

### GDB 7

GDB 7 support is provided in `Tools/gdb/libpython.py` in the python
source distribution. This is not included in the `-dbg` or main python
binary packages on Ubuntu.  This file is also left in the base build
directory of Python, named `python-gdb.py`.

Enabling in `~/.gdbinit`: 
```
add-auto-load-safe-path /home/erics/build/python3.5-3.5.2/ 
py sys.path.append('/home/erics/build/python3.5-3.5.2/Tools/gdb')
py import libpython

```

This adds support for some python types in GDB:

Without:
```
#2  0x00000000004adec9 in _PyCFunction_FastCallDict (kwargs=0x0, nargs=<optimized out>, args=0x93c950, func_obj=0x7ffff3c10630)
    at Objects/methodobject.c:234
234	            result = (*meth) (self, tuple);
(gdb) info locals
tuple = 0x7ffff3b87120
func = 0x7ffff3c10630
meth = 0x7ffff386a728 <_fill>
self = 0x7ffff6ada818
flags = <optimized out>
result = <optimized out>
(gdb) p *tuple
$3 = {ob_refcnt = 1, ob_type = 0x884a60 <PyTuple_Type>}
```

```
(gdb) bt
#0  PyImagingNew (imOut=0xb33d40) at _imaging.c:166
#1  0x00007ffff386a82e in _fill (self=0x7ffff6ada818, args=0x7ffff3b87120) at _imaging.c:617
#2  0x00000000004adec9 in _PyCFunction_FastCallDict (kwargs=0x0, nargs=<optimized out>, args=0x93c950, func_obj=0x7ffff3c10630)
    at Objects/methodobject.c:234
#3  _PyCFunction_FastCallKeywords (func=func@entry=0x7ffff3c10630, stack=stack@entry=0x93c950, nargs=<optimized out>, 
    kwnames=kwnames@entry=0x0) at Objects/methodobject.c:295
#4  0x000000000053e3ae in call_function (pp_stack=pp_stack@entry=0x7fffffffdd00, oparg=oparg@entry=3, kwnames=kwnames@entry=0x0)
    at Python/ceval.c:4798
#5  0x0000000000542c17 in _PyEval_EvalFrameDefault (f=<optimized out>, throwflag=<optimized out>) at Python/ceval.c:3284
#6  0x000000000053e015 in PyEval_EvalFrameEx (throwflag=0, f=0x93c7a8) at Python/ceval.c:718
#7  _PyEval_EvalCodeWithName (_co=0x7ffff6f78ae0, globals=globals@entry=0x7ffff6f59708, locals=locals@entry=0x0, 
    args=<optimized out>, argcount=2, kwnames=kwnames@entry=0x0, kwargs=0x7ffff70c8bb0, kwcount=0, kwstep=1, defs=0x7ffff681b290, 
    defcount=1, kwdefs=0x0, closure=0x0, name=0x7ffff70b6810, qualname=0x7ffff70b6810) at Python/ceval.c:4128
#8  0x000000000053e2c7 in fast_function (kwnames=0x0, nargs=<optimized out>, stack=<optimized out>, func=0x7ffff3aef730)
    at Python/ceval.c:4939
#9  call_function (pp_stack=pp_stack@entry=0x7fffffffdf60, oparg=oparg@entry=2, kwnames=kwnames@entry=0x0) at Python/ceval.c:4819
#10 0x0000000000542c17 in _PyEval_EvalFrameDefault (f=<optimized out>, throwflag=<optimized out>) at Python/ceval.c:3284
#11 0x000000000053e015 in PyEval_EvalFrameEx (throwflag=0, f=0x7ffff70c8a20) at Python/ceval.c:718
#12 _PyEval_EvalCodeWithName (_co=_co@entry=0x7ffff7050810, globals=globals@entry=0x7ffff70b1198, 
    locals=locals@entry=0x7ffff7050810, args=args@entry=0x0, argcount=argcount@entry=0, kwnames=kwnames@entry=0x0, kwargs=0x8, 
    kwcount=0, kwstep=2, defs=0x0, defcount=0, kwdefs=0x0, closure=0x0, name=0x0, qualname=0x0) at Python/ceval.c:4128
#13 0x000000000053ee43 in PyEval_EvalCodeEx (closure=0x0, kwdefs=0x0, defcount=0, defs=0x0, kwcount=0, kws=0x0, argcount=0, 
    args=0x0, locals=locals@entry=0x7ffff7050810, globals=globals@entry=0x7ffff70b1198, _co=_co@entry=0x7ffff7050810)
    at Python/ceval.c:4149
#14 PyEval_EvalCode (co=co@entry=0x7ffff7050810, globals=globals@entry=0x7ffff70991b0, locals=locals@entry=0x7ffff70991b0)
    at Python/ceval.c:695
#15 0x000000000042740f in run_mod (arena=0x7ffff70b1198, flags=0x7fffffffe240, locals=0x7ffff70991b0, globals=0x7ffff70991b0, 
    filename=0x7ffff6fa69b0, mod=0x969290) at Python/pythonrun.c:980
#16 PyRun_FileExFlags (fp=0x9372a0, filename_str=<optimized out>, start=<optimized out>, globals=0x7ffff70991b0, 
    locals=0x7ffff70991b0, closeit=1, flags=0x7fffffffe240) at Python/pythonrun.c:933
#17 0x000000000042763c in PyRun_SimpleFileExFlags (fp=0x9372a0, filename=<optimized out>, closeit=1, flags=0x7fffffffe240)
    at Python/pythonrun.c:396
#18 0x000000000043b975 in run_file (p_cf=0x7fffffffe240, filename=0x8f82b0 L"issue_2620.py", fp=0x9372a0) at Modules/main.c:338
#19 Py_Main (argc=argc@entry=2, argv=argv@entry=0x8f7010) at Modules/main.c:809
#20 0x000000000041dc52 in main (argc=2, argv=<optimized out>) at ./Programs/python.c:69
(gdb) py-bt
```



With:
```
#2  0x00000000004adec9 in _PyCFunction_FastCallDict (kwargs=0x0, nargs=<optimized out>, args=0x93c950, 
    func_obj=<built-in method fill of module object at remote 0x7ffff6ada818>) at Objects/methodobject.c:234
234	            result = (*meth) (self, tuple);
(gdb) info locals
tuple = ('RGB', (100, 64), 0)
func = 0x7ffff3c10678
meth = 0x7ffff386a728 <_fill>
self = <module at remote 0x7ffff6ada818>
flags = <optimized out>
result = <optimized out>
```

```
#0  PyImagingNew (imOut=0xb33d40) at _imaging.c:167
#1  0x00007ffff386a82e in _fill (self=<module at remote 0x7ffff6ada818>, args=('RGB', (100, 64), 0)) at _imaging.c:617
#2  0x00000000004adec9 in _PyCFunction_FastCallDict (kwargs=0x0, nargs=<optimized out>, args=0x93c950, 
    func_obj=<built-in method fill of module object at remote 0x7ffff6ada818>) at Objects/methodobject.c:234
#3  _PyCFunction_FastCallKeywords (func=func@entry=<built-in method fill of module object at remote 0x7ffff6ada818>, 
    stack=stack@entry=0x93c950, nargs=<optimized out>, kwnames=kwnames@entry=0x0) at Objects/methodobject.c:295
#4  0x000000000053e3ae in call_function (pp_stack=pp_stack@entry=0x7fffffffdd00, oparg=oparg@entry=3, kwnames=kwnames@entry=0x0)
    at Python/ceval.c:4798
#5  0x0000000000542c17 in _PyEval_EvalFrameDefault (f=<optimized out>, throwflag=<optimized out>) at Python/ceval.c:3284
#6  0x000000000053e015 in PyEval_EvalFrameEx (throwflag=0, 
    f=Frame 0x93c7a8, for file /home/erics/vpy361/lib/python3.6/site-packages/Pillow-4.3.0.dev0-py3.6-linux-x86_64.egg/PIL/Image.py, line 2263, in new (mode='RGB', size=(100, 64), color=0)) at Python/ceval.c:718
#7  _PyEval_EvalCodeWithName (_co=<code at remote 0x7ffff6f78ae0>, 
    globals=globals@entry={'__name__': 'PIL.Image', '__doc__': None, '__package__': 'PIL', '__loader__': <zipimport.zipimporter at remote 0x7ffff6f4b148>, '__spec__': <ModuleSpec(name='PIL.Image', loader=<zipimport.zipimporter at remote 0x7ffff6f4b148>, origin='/home/erics/vpy361/lib/python3.6/site-packages/Pillow-4.3.0.dev0-py3.6-linux-x86_64.egg/PIL/Image.py', loader_state=None, submodule_search_locations=None, _set_fileattr=True, _cached=None) at remote 0x7ffff6f71dd8>, '__builtins__': {'__name__': 'builtins', '__doc__': "Built-in functions, exceptions, and other objects.\n\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices.", '__package__': '', '__loader__': <type at remote 0x909418>, '__spec__': <ModuleSpec(name='builtins', loader=<type at remote 0x909418>, origin=None, loader_state=None, submodule_search_locations=None, _set_fileattr=False, _cached=None) at remote 0x7ffff70ca630>, '__build_class__': <built-in method __build_class__ of module object at remote 0x7ffff7e465e8>, '__import__': <built-in method...(truncated), 
    locals=locals@entry=0x0, args=<optimized out>, argcount=2, kwnames=kwnames@entry=0x0, kwargs=0x7ffff70c8bb0, kwcount=0, 
    kwstep=1, defs=0x7ffff681b290, defcount=1, kwdefs=0x0, closure=0x0, name='new', qualname='new') at Python/ceval.c:4128
#8  0x000000000053e2c7 in fast_function (kwnames=0x0, nargs=<optimized out>, stack=<optimized out>, 
    func=<function at remote 0x7ffff3aef730>) at Python/ceval.c:4939
#9  call_function (pp_stack=pp_stack@entry=0x7fffffffdf60, oparg=oparg@entry=2, kwnames=kwnames@entry=0x0) at Python/ceval.c:4819
#10 0x0000000000542c17 in _PyEval_EvalFrameDefault (f=<optimized out>, throwflag=<optimized out>) at Python/ceval.c:3284
#11 0x000000000053e015 in PyEval_EvalFrameEx (throwflag=0, f=Frame 0x7ffff70c8a20, for file issue_2620.py, line 10, in <module> ())
    at Python/ceval.c:718
#12 _PyEval_EvalCodeWithName (_co=_co@entry=<code at remote 0x7ffff7050810>, 
    globals=globals@entry=4198513420969916515817303439419953979824347301742789106172114332123381364613718940335931020105255659995471430970402158659953887164677159017819495708587307635513414206860475424125598971939551304434745448512308466513545318556705730291257286215616133621416869400099763968603741949341946581427524829373894687827964204922827560959399228057214652467732234267750694212138276139704386891615317965596537353962231590184148122300116257245912786836485179518668244188796641428699894551179013291458337233042370598150379055231423048771823916586915040542987124690695873981894870533453959989695926983735450601586434743056968486251180242504012252702304117996152511705580255459002228329601831343788551397411514246481292768931732675913163560628843227953414928044850241424869272709025986350261416691070819139813631824375974294951438290322394035759052705596013353933377932229610135407383834314811659079600770611659688017858864413167237386347011344148793946879889117346158800344941012840466439773780559844894128031141750352722505290195584172789603...(truncated), 
    locals=locals@entry=<code at remote 0x7ffff7050810>, args=args@entry=0x0, argcount=argcount@entry=0, kwnames=kwnames@entry=0x0, 
    kwargs=0x8, kwcount=0, kwstep=2, defs=0x0, defcount=0, kwdefs=0x0, closure=0x0, name=0x0, qualname=0x0) at Python/ceval.c:4128
#13 0x000000000053ee43 in PyEval_EvalCodeEx (closure=0x0, kwdefs=0x0, defcount=0, defs=0x0, kwcount=0, kws=0x0, argcount=0, 
    args=0x0, locals=locals@entry=<code at remote 0x7ffff7050810>, 
    globals=globals@entry=4198513420969916515817303439419953979824347301742789106172114332123381364613718940335931020105255659995471430970402158659953887164677159017819495708587307635513414206860475424125598971939551304434745448512308466513545318556705730291257286215616133621416869400099763968603741949341946581427524829373894687827964204922827560959399228057214652467732234267750694212138276139704386891615317965596537353962231590184148122300116257245912786836485179518668244188796641428699894551179013291458337233042370598150379055231423048771823916586915040542987124690695873981894870533453959989695926983735450601586434743056968486251180242504012252702304117996152511705580255459002228329601831343788551397411514246481292768931732675913163560628843227953414928044850241424869272709025986350261416691070819139813631824375974294951438290322394035759052705596013353933377932229610135407383834314811659079600770611659688017858864413167237386347011344148793946879889117346158800344941012840466439773780559844894128031141750352722505290195584172789603...(truncated), 
    _co=_co@entry=<code at remote 0x7ffff7050810>) at Python/ceval.c:4149
#14 PyEval_EvalCode (co=co@entry=<code at remote 0x7ffff7050810>, 
    globals=globals@entry={'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <SourceFileLoader(name='__main__', path='issue_2620.py') at remote 0x7ffff704ff98>, '__spec__': None, '__annotations__': {}, '__builtins__': <module at remote 0x7ffff7e465e8>, '__file__': 'issue_2620.py', '__cached__': None, 'os': <module at remote 0x7ffff7053e58>, 'Image': <module at remote 0x7ffff68246d8>, 'ImageFont': <module at remote 0x7ffff3aec318>, 'ImageDraw': <module at remote 0x7ffff3aecb88>, 'mode': 'RGB', 'size': (100, 64)}, 
```

And python backtraces:

```
(gdb) py-bt
 Traceback (most recent call first):
  <built-in method fill of module object at remote 0x7ffff68eeed8>
  File "/home/erics/vpy35-dbg/lib/python3.5/site-packages/Pillow-4.3.0.dev0-py3.5-linux-x86_64.egg/PIL/Image.py", line 2263, in new
    return Image()._new(core.fill(mode, size, color))
  File "issue_2620.py", line 10, in <module>
    im = Image.new(mode, size)
```

Caveats:

- The python source should match the version of python
being debugged or there will be issues. 

- The version of python embedded in gdb can be found with:
```
(gdb) py print(sys.version)
3.5.2 (default, Nov 17 2016, 17:05:23) 
[GCC 5.4.0 20160609]
```


### GDB 6

`/usr/share/doc/python[version]/gdbinit.gz`  Despite the documentation
in the package, these are in the versioned doc directory, not the
python directories. This is enabled by copying into your `.gdbinit`


## Digging in

### Small Test Case
### Segfault
### Not so small test case
### Power tools
 * break on next line of python
 * break before return to python
 * break on return from C function

## Namespaces

```sudo nsenter -t [pid on host] -p -m [cmd]
docker run -it --rm --pid=host myhtop
docker run -it --pid=container:my-redis my_strace_docker_image bash
```
(on ubuntu)
In the container namespace, `/proc/sys/kernel/yama/ptrace_scope = 1` only
allows direct children to be traced, so `gdb -p pid` is disallowed by
the kernel. In theory, we could run this docker with cap_ptrace, but
that's not going to help us attach to a process that's already
running. 

So, building a docker image and running:
```
docker run -ti --rm --pid=container:f839629ded6b -u root
pythonpillow/ubuntu-xenial-amd64-gdb /bin/bash
```
isn't quite going to cut it, even running as root. 

`--cap-add=SYS_PTRACE` doesn't work.
`--sysctl kernel.yama.ptrace_scope=0` doesn't work. 
```invalid argument "kernel.yama.ptrace_scope=0" for --sysctl: sysctl
'kernel.yama.ptrace_scope=0' is not whitelisted
```

Setting `sudo sh -c "echo 0 > /proc/sys/kernel/yama/ptrace_scope"` on
the host doesn't work, at least at the level of restarting the docker
run to do gdb. And it doesn't work when restarting the original to be
debugged container either. 

Running the docker gdb container as `--security-opt="apparmor=unconfined"
--cap-add=SYS_PTRACE` does allow it to attach to the process, but it
can't open the executable, since the two containers have different
mount namespaces. What I need to do is bind mount the differences over
the original. 

It is possible to attach from more recent GDB on from the host, at
least when using sudo. 


## Platform Variations
### pdb vs gdb


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
