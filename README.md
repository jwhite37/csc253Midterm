Some Fancy Title
=====
Code Fragment
```
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
