A 5 Minute Tutorial on Python Function Calls
=====
This tutorial will give you a quick overview of how Python internally handles function calls. The below code is a simple example of how to define and call a function in Python.
``` Python
def add(x, y):
    return x + y

print add(1,2)
```
As we know all Python code is compiled into bytecode to be executed by the interpreter, the below gives this bytecode. Notice that the first opcode is loading up a code object, this in itself is another fragment of bytecode that will be passed into the evaluation function as the program is executed. More on frames, functions, and code is described below.
```
python -m dis test.py
  1           0 LOAD_CONST               0 (<code object add at 0x6ffffe2fa30, file "test.py", line 1>)
              3 MAKE_FUNCTION            0
              6 STORE_NAME               0 (add)

  4           9 LOAD_NAME                0 (add)
             12 LOAD_CONST               1 (1)
             15 LOAD_CONST               2 (2)
             18 CALL_FUNCTION            2
             21 PRINT_ITEM
             22 PRINT_NEWLINE
             23 LOAD_CONST               3 (None)
             26 RETURN_VALUE
```
Frame vs Function vs Code
====

First of all, all three of these are PyObjects. They are closely related to each other. 
Function is Code plus some closure, and Frame is the runtime instance of a function.
What really gets executed at the end, is the Frame. Function is like a prototype of 
Frame, and Frame is the instanciated version of Function.

The most fundamental building block is the Code.
It looks like this:
``` C++
/* Bytecode object */
typedef struct {
    PyObject_HEAD
    int co_argcount;		/* #arguments, except *args */
    int co_nlocals;		/* #local variables */
    int co_stacksize;		/* #entries needed for evaluation stack */
    int co_flags;		/* CO_..., see below */
    PyObject *co_code;		/* instruction opcodes */
    PyObject *co_consts;	/* list (constants used) */
    PyObject *co_names;		/* list of strings (names used) */
    PyObject *co_varnames;	/* tuple of strings (local variable names) */
    PyObject *co_freevars;	/* tuple of strings (free variable names) */
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
    /* The rest doesn't count for hash/cmp */
    PyObject *co_filename;	/* string (where it was loaded from) */
    PyObject *co_name;		/* string (name, for reference) */
    int co_firstlineno;		/* first source line number */
    PyObject *co_lnotab;	/* string (encoding addr<->lineno mapping) See
				   Objects/lnotab_notes.txt for details. */
    void *co_zombieframe;     /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
} PyCodeObject;
```
Code in the source code is represented by this PyCodeObject. 
Notice that to PyCodeObject’s knowledge, there is nothing like global. All it knows at the moment is
its local variables, even these local ones are not yet bound to some value, they are just symbols.

The interesting part here is the `PyCodeObject.co_code`, which is the actual byte code of the add
function in the source.

```
          0 LOAD_FAST           0 (0)
          3 LOAD_FAST           1 (1)
          6 BINARY_ADD
          7 RETURN_VALUE
```

The concept of Function builds upon the concept of Code. It is equivalently Code plus Closure.
(i.e. Function = Code + Closure).

It looks like this:
``` C++
typedef struct {
    PyObject_HEAD
    PyObject *func_code;	/* A code object */
    PyObject *func_globals;	/* A dictionary (other mappings won't do) */
    PyObject *func_defaults;	/* NULL or a tuple */
    PyObject *func_closure;	/* NULL or a tuple of cell objects */
    PyObject *func_doc;		/* The __doc__ attribute, can be anything */
    PyObject *func_name;	/* The __name__ attribute, a string object */
    PyObject *func_dict;	/* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist;	/* List of weak references */
    PyObject *func_module;	/* The __module__ attribute, can be anything */

    /* Invariant:
     *     func_closure contains the bindings for func_code->co_freevars, so
     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
     */
} PyFunctionObject;
```

Notice that here, `PyFunctionObject.func_global` kicks in. Function knows what the global is to some Code. Code itself 
does not need to worry about its globals.  

Frame is pretty much like the runtime representation of a Function.

Python Internals
=====

Now that we have some background on Frames, Functions, and Code, let's take a look at how Python internally would execute our function definition and execution. This can really be broken up into two parts centering around the `MAKE_FUNCTION` and `CALL_FUNCTION` opcodes, which in our bytecode is neatly broken up corresponding to line 1 and 4 in the original Python code.

```
  1           0 LOAD_CONST               0 (<code object add at 0x6ffffe2fa30, file "test.py", line 1>)
              3 MAKE_FUNCTION            0
              6 STORE_NAME               0 (add)
```
We've seen the `LOAD_CONST` and `STORE_NAME` before, and these work the same way, loading a PyObject* onto the value stack and storing an object in a dictionary. The key to this entire process is the `MAKE_FUNCTION` opcode, specifically the line `x = PyFunction_New(v, f->f_globals);`. Combined with the Load and Store opcodes you can see right away due to how pythons internals are written what's happening, we're creating a Function Object using the Code Object we just loaded and the globals from the frame we're currently executing. You can see this functions full code in `Objects\funcobject.c` to get a full view of implementation, but overall we're simply initializing a Function Object with a pointer to the Code Object so we can at a later point actually execute the bytecode when calling the function.

So now we're left with our Function Oblect on top of the value stack, now it's time to actually execute the function passing in our parameters and do some work with it.

```
  4           9 LOAD_NAME                0 (add)
             12 LOAD_CONST               1 (1)
             15 LOAD_CONST               2 (2)
             18 CALL_FUNCTION            2
             21 PRINT_ITEM
             22 PRINT_NEWLINE
             23 LOAD_CONST               3 (None)
             26 RETURN_VALUE
```

LOAD_NAME - Loading our FunctionObject by dictionary lookup.
LOAD_CONST - Loading our constants via dictionary lookup.

CALL FUNCTION - First calling macro PCALL(PCALL_ALL)  - these macros only get used if we have CALLPROFILES enabled, this allows for storing function calls we've made before, this is disabled for us so I think we can ignore them.
- calling call_function passing in the stack pointer and oparg of 2
- PyCFunctions are wrapped callable functions (so wrappers for a c function) so we take the 'else' branch here.
- PyMethod_Check = 'false' since our function isn't a PyMethodObject (It's a PyFunctionObject), so we go even further to the fast function call.
- the fast_function grabs the code object, globals, and defaults (I don't think we have defaults in this one), need to figure out co->co flags.
