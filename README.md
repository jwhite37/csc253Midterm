Some Fancy Title
=====
Code Fragment
``` Python
def add(x, y):
    return x + y

print add(1,2)
```
Dissassembled Bytecode
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


LOAD_CONST - Loading up the code object that defines our function. (The actual opcodes that get executed for the function)

MAKE_FUNCTION - Going to funcobject.c :Mention something on garbage collection (possibly). Here we're creating a function object (using the code object we just retreived and the globals so the function can have access to any globals) and passing it back out to ceval to store after setting the Garbage Collection tracking for the object. Pushing that function object onto the value stack.

- Possibly looking into the module thing a bit more here in funcobject.c

STORE_NAME - Store the FunctionObject into the name (add) Locals is a dictionary so we can easily store this reference in the locals with name 'add', which is one of the names from our code object.

LOAD_NAME - Loading our FunctionObject by dictionary lookup.
LOAD_CONST - Loading our constants via dictionary lookup.

CALL FUNCTION - First calling macro PCALL(PCALL_ALL)  - these macros only get used if we have CALLPROFILES enabled, this allows for storing function calls we've made before, this is disabled for us so I think we can ignore them.
- calling call_function passing in the stack pointer and oparg of 2
- PyCFunctions are wrapped callable functions (so wrappers for a c function) so we take the 'else' branch here.
- PyMethod_Check = 'false' since our function isn't a PyMethodObject (It's a PyFunctionObject), so we go even further to the fast function call.
- the fast_function grabs the code object, globals, and defaults (I don't think we have defaults in this one), need to figure out co->co flags.
