A 5 Minute Tutorial on Python Function Calls
=====
This tutorial gives you a quick overview of how Python internally handles function calls.

The python source we are going to examine is:
``` Python
def add(x, y):
    return x + y

print add(1,2)
```
Correspoinding bytecode for the above source is:
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

Notice that the first bytecode (`0 LOAD_CONST`) loads an code object in. What
gets loaded is the code object of the `add` function in the python source.

The disassembled bytecode of that code object is:
```
          0 LOAD_FAST           0 (0)
          3 LOAD_FAST           1 (1)
          6 BINARY_ADD
          7 RETURN_VALUE
```

Python interpreter evaluates frame, which is something sort of like a instanciated function.
Function always have a code part in it. 

Frame vs Function vs Code
====

First of all, all three of these are PyObjects. They are closely related to each other. 
Function is Code plus some closure, and Frame is the runtime instance of a function.
What really gets executed at the end, is the Frame. Function is like a prototype of 
Frame, and Frame is the instantiated version of Function.

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
Notice that to the PyCodeObject’s knowledge, there is nothing like global nor closure. All it knows at 
the moment is its local variables, even these local ones are not yet bound to some value, they are 
just symbols.

The interesting part here is the `PyCodeObject.co_code`, which, in our case,is the actual byte code
of the add function. More precisely, the `co_varnames` are local symbols ( not necessaraly bound ).
In our case are `x` and `y`. 

```
          0 LOAD_FAST           0 (0)
          3 LOAD_FAST           1 (1)
          6 BINARY_ADD
          7 RETURN_VALUE
```

The concept of Function builds upon the concept of Code. It is equivalently Code plus the context
in which it is created. By context, I really mean `func_closure` and `func_globals`. Notice though, 
if we do not write nested `def some_function_name():` in the source, the `func_closure` is simply `NULL`.
However, when we have nested function and have function as return value, we will then invoke `MAKE_CLOSURE`
to set the `func_closure` so as to remember the context in which the returned function is created. 

For instance, here is a block of code where the `fun` inside `f_factory` is returned, the `fun` here 
will have its `fun.func_closure` set with tuple `(x, y, z)` to the value when `fun` is created. 

```
def add(x, y):
    return x + y

def f_factory(x, y):
    z = x*y
    def fun():
        return add(x, y) + z
    return fun

x = f_factory(1,3)
y = f_factory(2,3)
print x(), y()
```

Therefore, please remember that:
`Function = Code + func_closure + func_global`

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

Frame is pretty much like the Functions and their contexts all together as a whole that gets executed by the Python VM. 

Python Internals
=====

Now that we have some background on Frames, Functions, and Code, let's take a look at how Python internally would execute our function definition and execution. This can really be broken up into two parts centering around the `MAKE_FUNCTION` and `CALL_FUNCTION` opcodes, which in our bytecode is neatly broken up corresponding to line 1 and 4 in the original Python code.

```
  1           0 LOAD_CONST               0 (<code object add at 0x6ffffe2fa30, file "test.py", line 1>)
              3 MAKE_FUNCTION            0
              6 STORE_NAME               0 (add)
```
We've seen the `LOAD_CONST` and `STORE_NAME` before, and these work the same way, loading a PyObject* onto the value stack and storing an object in a dictionary. The key to this entire process is the `MAKE_FUNCTION` opcode, specifically the line `x = PyFunction_New(v, f->f_globals);`. Combined with the Load and Store opcodes you can see right away due to how pythons internals are written what's happening, we're creating a Function Object using the Code Object we just loaded and the globals from the frame we're currently executing. You can see this functions full code in `Objects\funcobject.c` to get a full view of implementation, but overall we're initializing a Function Object with a pointer to the Code Object so we can at a later point actually execute the bytecode when calling the function.

So now we're left with our Function Object on top of the value stack, now it's time to actually execute the function passing in our parameters and do some work with it.

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
The first three opcodes being processed by the interpreter are pretty straight foward, in short we're loading our Function Object and two Integer Objects onto the stack so we can execute the function with them.

The main interesting bit of the above is in executing `CALL_FUNCTION` when the following call `x = call_function(&sp, oparg);` is made. Notice that we pass in the current stack pointer from the executing frame by reference, this will come into play later on.

The implementation of `call_function` can be found on line 4062 of ceval.c, first we're defining some values, we've added in some comments to help you understand what's being initialized here inline with the actual elements being setup.
``` C
   int na = oparg & 0xff; 			//Get the number of arguments
   int nk = (oparg>>8) & 0xff;			//Keyword number occupies higher order bits, 0 in our case
   int n = na + 2 * nk;				//n is the 'space' on the value stack that concerns this call
   PyObject **pfunc = (*pp_stack) - n - 1;	//Get a pointer to the function object
   PyObject *func = *pfunc;			//Retrieve the actual function object
```

The interpreter does a couple of checks immediately in order to find out if the function is a Python wrapper for a c function or a Method object since these are the most common calls made, our function is just a straight forward PyFunctionObject, so we execute the branch of code that calls `x = fast_function(func, pp_stack, n, na, nk);`. 

In the call to `fast_function`, the interpreter first sets things up, commented below.

``` C
    PyCodeObject *co = (PyCodeObject *)PyFunction_GET_CODE(func);	//Get a pointer to the Code Object
    PyObject *globals = PyFunction_GET_GLOBALS(func);			//Get a pointer to the functions globals
```

Once the `fast_function` method is setup, the code takes the very first branch since `argdefs == NULL && co->co_argcount == n && nk==0 && co->co_flags == (CO_OPTIMIZED | CO_NEWLOCALS | CO_NOFREE)` is a true statement. Notice we're checking to see how many arguments the Code Object has against how many arguments are on the stack; this is what makes this execution 'fast' as everything that's needed to complete the call is actually already loaded up and ready to go and we can access it all by pointer arithmatic.

The interesting bit of code here remaining in this function is the following.
``` C
        PyFrameObject *f;
	.
	.
	.
        f = PyFrame_New(tstate, co, globals, NULL);

        fastlocals = f->f_localsplus;
        stack = (*pp_stack) - n; 			//Getting a direct pointer to the very first argument

        for (i = 0; i < n; i++) {
            Py_INCREF(*stack);
            fastlocals[i] = *stack++;
        }
        retval = PyEval_EvalFrameEx(f,0);
```

The interpreter creates a brand new Frame Object using the Code Object and Globals. It then gets a pointer to the locals for the frame, and for each of the arguments on the stack (there are 2 in our case) places them into this locals array. This action is what allows for the `LOAD_FAST` opcodes in the Code Object to do their thing, instead of a dictionary lookup we can make an atomic array read operation to get our arguments.

Then the frame is evaluated using the Code Object, both arguments are added and the remaining value is left on top of the value stack. Notice that since we've passed in the address of the value stack from frame one all of this is done on the exact same value stack. The one opcode needing a bit of explanation here is the `RETURN_VALUE`, which grabs the top of the stack and executes a `fast_block_end` which kicks our result out to the caller. This is how the result of addition gets back out to our original frame.

So now we find ourselves back in the `call_function` routine after returning the result from `fast_function`, the Interpreter executes this bit of code below to clean up the arguments and function still sitting on the value stack since they're no longer needed by simply running the stack pointer back to the pointer to the function we created earlier. This is why it was important to pass in the `stack_pointer` by reference, it gives the ability to do the cleanup work prior to returning to the `CALL_FUNCTION` opcode execution.
```
    /* Clear the stack of the function object.  Also removes
       the arguments in case they weren't consumed already
       (fast_function() and err_args() leave them on the stack).
     */
    while ((*pp_stack) > pfunc) {
        w = EXT_POP(*pp_stack);
        Py_DECREF(w);
    }
    return x;
```

Following this we execute the remaining opcodes after resetting the `stack_pointer` in `ceval.c` to the one returned to us by the reference passing in `call_function`. The remaining opcodes are fairly straight forward and not particularly interesting when it comes to function calls. We store our result onto the value stack, print out the value, load nothing onto the value stack and exit the frame!
