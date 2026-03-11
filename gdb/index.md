# GDB Cheat Sheet


* Configuration
- Save history
#+begin_src conf
set history save on
#+end_src

* Args
#+begin_src shell
gdb --args ./d8 --allow-natives-syntax jit.js --single-threaded
#+end_src

* Print
** history command
#+begin_src gdb
C-r (reverse-i-search)`':
#+end_src

** address
#+begin_src gdb
  x/nfu addr

  n: How many units to print (default 1).

  f: Format character (like print)
      o - octal
      x - hexadecimal
      d - decimal
      u - unsigned decimal
      t - binary
      f - floating point
      a - address
      c - char
      s - string
      i - instruction

      u: Unit.
      b: Byte,
      h: Half-word (two bytes)
      w: Word (four bytes)
      g: Giant word (eight bytes))
#+end_src

** print hex
#+begin_src gdb
p/x variable
#+end_src


** show all functions

#+begin_src gdb
info functions [regexp]
#+end_src

** show address info
#+begin_src gdb
info symbol addr
#+end_src

* Kernel Debugging
#+begin_src
(gdb) apropos lx
function lx_current -- Return current task
function lx_module -- Find module by name and return the module variable
function lx_per_cpu -- Return per-cpu variable
function lx_task_by_pid -- Find Linux task by PID and return the task_struct variable
function lx_thread_info -- Calculate Linux thread_info from task variable
lx-dmesg -- Print Linux kernel log buffer
lx-lsmod -- List currently loaded modules
lx-symbols -- (Re-)load symbols of Linux kernel and currently loaded modules
#+end_src

* Radare2

** analysis

| Command    | Description      |
| aa         | analyze all      |
| afl        | list functions   |
| s sym.main | seek to function |

** inspecting

| Command | Description          |
| pdb     | basic block          |
| pdf     | function disassembly |
| afa     | function arguments   |
| afv     | function variables   |
| af      | analyze function     |

** graph output

| Command | Description           |
| agfv    | Interactive Ascii Art |
| agfd    | Graphviz dot          |

