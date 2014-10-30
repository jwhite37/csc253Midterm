A Tutorial on Python Function Calls
=====
This tutorial gives you a quick overview of how Python handles function calls.

The Python program we are going to examine is:
```python
def add(x, y):
    return x + y

print add(1,2)
```
Corresponding bytecode for the above source is:
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

The disassembled bytecode of the code object on line 1 above is:
```python
          0 LOAD_FAST           0 (0)
          3 LOAD_FAST           1 (1)
          6 BINARY_ADD
          7 RETURN_VALUE
```

Bytecode Walkthrough
====================
Starting from the function definition:
```python
python -m dis test.py
  1           0 LOAD_CONST               0 (<code object add at 0x6ffffe2fa30, file "test.py", line 1>)
              3 MAKE_FUNCTION            0          //Elaborated in later section MAKE_FUNCTION Walkthrough
              6 STORE_NAME               0 (add)
```
The Python interpreter makes a `PyCodeObject` and loads it to the value stack. 

Then it creates a `PyFunctionObject` out of the `PyCodeObject` on the value stack and leaves a `PyFunctionObject` on 
the value stack.

Then it binds the name `add` with the `PyFunctionObject` and pops the `PyFunctionObject` off the value stack.

At the moment of calling the function:
```python
  4           9 LOAD_NAME                0 (add)
             12 LOAD_CONST               1 (1)
             15 LOAD_CONST               2 (2)
             18 CALL_FUNCTION            2          //Elaborated in later section CALL_FUNCTION Walkthrough
             21 PRINT_ITEM
             22 PRINT_NEWLINE
             23 LOAD_CONST               3 (None)
             26 RETURN_VALUE
```
The Python interpreter loads the function name ( in our case `add` ) and two arguments ( in our case `1` and `2` ) to 
the value satck. 

Then the interpreter follows `CALL_FUNCTION` to make a `PyFrameObject` out of the `PyFunctionObject` with the name `add` 
and evaluates it in `ceval.c`'s main loop. The bytecode finishes by putting a returned Object on to the value stack. In 
our case `CALL_FUNCTION` puts a `PyIntObject` (whose value slot is set to 3) on to the value stack. 

The `PRINT_ITEM` bytecode tells the Python Interpreter to pop the `PyIntObject` off of the value stack and 
print its value to standard output and `PRINT_NEWLINE` prints out a newline character. 

Python interpreter then loads in the `None` PyObject to the value stack so that it could be used by the following 
`RETURN_VALUE` bytecode.

At last, the `RETURN_VALUE` bytecode instructs the Python interpreter to return the top of the stack object
(which in our case is the PyIntObject we loaded in from `LOAD_CONSTANT`) to the caller. 

The most interesting parts of the above bytecode are `MAKE_FUNCTION` and `CALL_FUNCTION`. We are going to elaborate on them individually below.

`MAKE_FUNCTION` Walkthrough
===========================
The key to this entire process is the `MAKE_FUNCTION` opcode, whose logic is like this:
```C
case MAKE_FUNCTION:
     v = POP();	                                // PyCodeObject
     x = PyFunction_New(v, f->f_globals);       // Make a function out of code object and global variables
     ...                                        
     PUSH(x);                                   // Put PyFunctionObject back to the value stack
     break;
```
The line `x = PyFunction_New(v, f->f_globals);` is the most interesting one. Combined with the Load and Store opcodes you can see right away due to how pythons internals are written what's happening, we're creating a Function Object using the Code Object we just loaded as well as the globals from the frame we're currently executing. You can see this functions full code in `Objects\funcobject.c` to get a full view of implementation, but overall we're initializing a Function Object with a pointer to the Code Object so we can at a later point actually execute the bytecode when calling the function.


`CALL_FUNCTION` Walkthrough
===========================

So now we're left with our Function Object on top of the value stack, now it's time to actually execute the function passing in our parameters and do some work with it.

The main interesting bit of the above is in executing `CALL_FUNCTION` when the following call `x = call_function(&sp, oparg);` is made. Notice that we pass in the current stack pointer from the executing frame by reference, this will come into play later on.

The implementation of `call_function` is the following:
```C
static PyObject *
call_function(PyObject ***pp_stack, int oparg)
{
    int na = oparg & 0xff; 			            //Get the number of arguments
    int nk = (oparg>>8) & 0xff;			        //Keyword number occupies higher order bits, 0 in our case
    int n = na + 2 * nk;				        //n is the 'space' on the value stack that concerns this call
    PyObject **pfunc = (*pp_stack) - n - 1;	    //Get a pointer to the function object
    PyObject *func = *pfunc;			        //Retrieve the actual function object
    PyObject *x, *w;
    ...
    if (PyCFunction_Check(func) && nk == 0) {
        ...
    } else {
        ...
        if (PyFunction_Check(func))
            x = fast_function(func, pp_stack, n, na, nk);       // We end up calling this
        else
            x = do_call(func, pp_stack, na, nk);
        READ_TIMESTAMP(*pintr1);
        Py_DECREF(func);
    }
    ...                                                         // Some clean up 
    }
    return x;
}
```
First we're defining some values, these values are used to figure out the position the stack pointer should point to based on
number of arguments for the function object as well as getting a handle to our `PyFunctionObject`.

The interpreter does a couple of checks immediately in order to find out if the function is a Python wrapper for a c function or a Method object since these are the most common calls made, our function is just a straight forward `PyFunctionObject`, so we execute the branch of code that calls `x = fast_function(func, pp_stack, n, na, nk);` which is also in `ceval.c`. 

In the call to `fast_function`, the interpreter first sets things up, commented below.

```C
    PyCodeObject *co = (PyCodeObject *)PyFunction_GET_CODE(func);	//Get a pointer to the Code Object
    PyObject *globals = PyFunction_GET_GLOBALS(func);			    //Get a pointer to the functions globals
```

Once the `fast_function` method is setup, the code takes the very first branch since `argdefs == NULL && co->co_argcount == n && nk==0 && co->co_flags == (CO_OPTIMIZED | CO_NEWLOCALS | CO_NOFREE)` is a true statement. Notice we're checking to see how many arguments the Code Object has against how many arguments are on the stack; this is what makes this execution 'fast' as everything that's needed to complete the call is actually already loaded up and ready to go and we can access it all by pointer arithmatic.

Now we get to the bit of code that actually executes our function.
```C
        PyFrameObject *f;
        ...
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

Then the Frame Object is evaluated through calling the `PyEval_EvalFrameEx` method (which is the same method that's been evaluating the initial main Frame Object). Both arguments are added and the remaining value is left on top of the value stack through a `BINARY_ADD`. Notice that since we've passed in the address of the value stack from the initial frame all of this is done on the exact same value stack. What essentially is happening here is the main frame transfers control of the stack to the new frame, and once this second frame finishes it returns control of the stack with modifications from executing the function code to the initial frame.

The one opcode needing a bit of explanation here is the `RETURN_VALUE`, which grabs the top of the stack and executes a `fast_block_end` which kicks our result out to the caller. This is how the result of addition gets back out to our original frame.

So now we find ourselves back in the `call_function` routine after returning the result from `fast_function`, the Interpreter executes this bit of code below to clean up the arguments and function still sitting on the value stack since they're no longer needed by simply running the stack pointer back to the pointer to the function we created earlier. This another reason why it was important to pass in the `stack_pointer` by reference, it gives the ability to do the cleanup work prior to returning to the `CALL_FUNCTION` opcode execution.
```C
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

Extended Discussion: PyFrameObject vs PyFunctionObject vs PyCodeObject
======================================================================

First of all, all three of these are PyObjects and are closely related to each other. 
Function is Code plus some closure, and Frame is the runtime instance of a function.
What really gets executed at the end, is the Frame. A Function is like a schema for a
Frame, and a Frame is an instantiated version of a Function.

The most fundamental building block is the Code Object.
It looks like this:
```C
/* Bytecode object */
typedef struct {
    PyObject_HEAD
    ...
    PyObject *co_code;		/* instruction opcodes */
    ...
    PyObject *co_varnames;	/* tuple of strings (local variable names) */
    ...                     //Some other stuff
} PyCodeObject;
```
A Code Object in the source code is represented by this PyCodeObject. 
Notice that to the PyCodeObject’s knowledge, there is nothing like global nor closure. All it knows at 
the moment is its local variables, even these local ones are not yet bound to some value, they are 
just symbols.

The interesting part here is the `PyCodeObject.co_code`, which, in our case is the actual byte code
of the add function. More precisely, the `co_varnames` are local symbols ( not necessaraly bound ).
In our case are `x` and `y`. 

```python
          0 LOAD_FAST           0 (0)
          3 LOAD_FAST           1 (1)
          6 BINARY_ADD
          7 RETURN_VALUE
```

The concept of a Function Object builds upon the concept of a Code Object. It is equivalently a Code Object plus the context
in which it is created. By context, I really mean `func_closure` and `func_globals`. Notice though, 
if we do not write nested `def some_function_name():` in the source, the `func_closure` is simply `NULL`.
However, when we have nested functions or have a function as a return value, we will then invoke `MAKE_CLOSURE`
to set the `func_closure` so as to remember the context in which the returned function is created. 

For instance, here is a block of code where the `fun` inside `f_factory` is returned, the `fun` here 
will have its `fun.func_closure` set with tuple `(x, y, z)` to the value when `fun` is created. 

```python
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
```C
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

Notice that here, `PyFunctionObject.func_global` kicks in. A Function Object knows what the global space is for some Code Object. Code Objects themselves need not worry about its globals.  

Frame is pretty much like the Functions and their contexts all together as a whole that gets executed by the Python VM. 


