# Proposal to add assignment and LOAD overloading

This pre-pep aims to introduce the ability for a class to specify what will happen when a named instance is
loaded by the interpreter, or when the name is attempted to be re-bound. This is similar to the behavior of a
property field on a class, but for an instance the class itself. Variables that implement these behaviors will
be referred to as cloaking variables in this document, as what the user sees cloaks the underlying object.
Towards the bottom of this document is a set of behaviors this functionality would add, but is by no means
exhaustive.  The following section describes some technical aspects of the implementation in cpython.

A preliminary working implementation of this behavior has been made against cpython 3.7. The code can be found
[here](https://github.com/natelust/cpython/tree/cloakingVars) (The repo should be on the cloakingVars branch.

This code was tested on an 18.04 Ubuntu linux installation with CFLAGS="-g -O0"

## Implementation details
This change adds two new entries to the type struct named `tp_setself` and `tp_getself`. These names are entirely
negotiable, but naming is hard and they had nice symmetry with all the other get and set methods.

Under normal circumstances, these fields would be NULL. If, however, the user builds a class and adds the
magic methods `__setself__` or `__getself__` assignment and LOAD are overloaded respectively. This is
technically achieved by checking if the corresponding type fields are null when doing either operation, and
calling the defined method if they are not.

These changes have the effect of increasing the size of the type struct by the width of two pointers. The
runtime penalty on code that does not use this behavior is only an if statement on "get" and "set" on the
corresponding dict (or array in the case of fastlocals). This statement checks if a pointer is null, so
very minimal. All other changes are put inside this if block.

This only modifies the behaviors of dicts that are used as namespace containers. Dicts that are used in the
normal course of a user program retain the existing behavior. This is both because the operation can be seen
as acting on the dict itself, and that this is likely to be the least confusing behavior. This behavior is the
same for all container types that are not "namespaces". This means that a cloaking variable in a dictionary
will be seen as a cloaking variable and not what the cloaking variable implements. Though technically
namespaces are the same as the dict type, in practice they appear differently to users.

As of the time of writing this document, this behavior does not extend to `__slots__` and or closure
variables. These retain the existing look-up behavior, i.e. the user will see the cloaking variable, and not
what it normally would present. There is no technical reason for this, but is simply not implemented yet.

## Impacts on existing code
There is almost no impact on existing code. The only exception is to classes that have defined `__getself__`
and `__setset__` methods for some functionality.

## Work to do
This is a working implementation of the new behavior that will not have an impact on existing code, but some
further work is required. This includes:

* Run extensive tests, and have more eyes for code review
* Verify all refcounting is handled correctly.
* Add in support for slots and closures
* Create builtins for working with these types of objects
* Write documentation

## Demo
Example python code which makes use of cloaking variable behavior. The output from this code can be found in
the section following the code.

```python
# These are "builtin" functions for working with cloaked variables                                                                
# They may be implemented in c at some point to avoid the inspect
# module, but this is pragmatic for now


def getcloaked(name):
    '''
    Retrieves the object underlying a cloaked variable
    var: str
        Variable name to look up
    '''
    import inspect
    outer = inspect.stack()
    if len(outer) == 0:
        ns = outer[0]
    else:
        ns = outer[1]
    return ns.frame.f_locals[name]


def setcloaked(name, value):
    '''
    Reassigns the variable associated with "name" to a new value,
    by passing any __setself__ defined on the cloaked variable
    var: str
        Variable name to look up
    '''
    import inspect
    outer = inspect.stack()
    if len(outer) == 0:
        ns = outer[0]
    else:
        ns = outer[1]
    ns.frame.f_locals[name] = value


def cloaksset(var, deep=1):
    ''' 
    Returns true if variable cloaks assignment
    var: str
        Variable name to look up
    '''
    import inspect
    outer = inspect.stack()
    if len(outer) == 0:
        ns = outer[0]
    else:
        ns = outer[deep]
    return hasattr(ns.frame.f_locals[var], '__setself__')


def cloaksget(var, deep=1):
    '''
    Returns true if the variable cloaks the LOAD operation
    var: str
        Variable name to look up
    '''
    import inspect
    outer = inspect.stack()
    if len(outer) == 0:
        ns = outer[0]
    else:
        ns = outer[deep]
    return hasattr(ns.frame.f_locals[var], '__getself__')
                                                                                                                                  

def iscloaked(var):
    '''
    Returns True if the variable cloaks LOAD or assignment
    var: str
        Variable name to look up
    '''
    return cloaksset(var, deep=2) or cloaksget(var, deep=2)


class HistoricVar:
    def __init__(self, start):
        self.var = start
        self.history = []

    def __repr__(self):
        return "This is a HistoricVar"

    def __getself__(self):
        return self.var

    def __setself__(self, value):
        self.history.append(self.var)
        self.var = value

    def rollback_n(self, n):
        if n > len(self.history):
            raise ValueError("Can't roll back before history started")
        for i in range(n-1):
            self.history.pop()

        self.var = self.history.pop()

    def get_history(self):
        return list(self.history)


print("Demoing a variable with history:")
print()
g = HistoricVar(2)
g = 12
g = "hello world"
g = [1, 2, 3]
print(f"The current value of g is {g}")
his = getcloaked('g').get_history()
print(f"The history of g is {his}")

# Roll the variable state back
print("Rolling back the history on g")
getcloaked('g').rollback_n(2)
print(f"The current value of g is {g}")
his = locals()['g'].get_history()
print(f"The history of g is {his}")
print()
print()

                                                                                                                                 
# An Example of a variable that is writes its contents to disk
# when assigned to


class FileBackedVar:
    def __init__(self, filename, starting):
        self.file = open(filename, 'wb')
        self.fileOpen = True
        self.value = starting
        self.__setself__(starting)

    def __getself__(self):
        return self.value

    def __setself__(self, value):
        import pickle  # noqa: F811
        self.value = value
        if self.fileOpen:
            self.file.seek(0)
            pickle.dump(value, self.file)
            self.file.truncate()

    def close_file(self):
        if self.fileOpen:
            self.file.close()
            self.FileOpen = False


print("Demoing a variable that syncs to disk")
print()
print("Creating a new file backed variable, with value 'hello world'")
fileVar = FileBackedVar('exampleFileVar', "hello world")
print("Reassigning the value to 'Brave new world'")
fileVar = "Brave new world"
print("Close the backing file")
getcloaked('fileVar').close_file()

with open('exampleFileVar', 'rb') as f:
    import pickle
    print("Load back in the saved var")
    value = pickle.load(f)
    print(f"The file var stored the value {value}")

print()
print()

# An implementation of Context Variables                                                                                          


class Context:
    declaredContextVars = {}

    def __init__(self):
        self.context_dict = {}

    def run(self, goer):
        for val in self.declaredContextVars.values():
            getcloaked('val').setcontext(self.context_dict)
        goer()


class ContextVar:
    def __init__(self, varname, default):
        Context.declaredContextVars[varname] = self
        self.default = default
        self.varname = varname

    def __getself__(self):
        try:
            retval = self.ctx.get(self.varname, self.default)
            return retval
        except Exception:
            return self.default

    def __setself__(self, value):
        try:
            self.ctx[self.varname] = value
        except Exception:
            raise TypeError("Can't set Context variable outside context")

    def setcontext(self, ctx):
        self.ctx = ctx


context1 = Context()
context2 = Context()
convar = ContextVar('convar', "hello world")


def set_context():
    global convar
    convar = 1


def get_context():
    print(convar)


print("Demoing an implementation of context variables:")
print()
print("Setting the context variable in context 1")
context1.run(set_context)
print("Printing the context variable in context 1")
context1.run(get_context)
print("Printing the context variable in context 2,"
      " it has the default value")
context2.run(get_context)
print()
print()

# Constants                                                                                                                       


class Constant:
    def __init__(self, wrapped):
        self.wrapped = wrapped

    def __getself__(self):
        return self.wrapped

    def __setself__(self, value):
        raise TypeError("Constant variables can't be reassigned")


CRITICAL_NUMBER = Constant(100)

print("Demoing cost variables:")
print("The declared constant is:")
print(CRITICAL_NUMBER)
print("The type of the declared constant is (i.e. the cloaking type):")
print(type(CRITICAL_NUMBER))
print("The real type is:")
print(type(getcloaked('CRITICAL_NUMBER')))

print("Attempting to reassign throws an error:")
try:
    CRITICAL_NUMBER = 105
except TypeError as e:
    print(e)
print()
print()


# Instance properties                                                                                                             

class InstanceProperty:
    def __init__(self, wrapped, getter, setter=None):
        self.wrapped = wrapped
        self.getter = getter
        self.setter = setter

    def __getself__(self):
        return self.getter(self.wrapped)

    def __setself__(self, value):
        if self.setter:
            return self.setter(self.wrapped, value)


class MachineState:
    def __init__(self):
        self._fields = {}

    def add_input(self, name, start):
        def getter(slf):
            return slf._fields[name]

        def setter(slf, value):
            '''
            the state of a machine part can only be above zero or below
            100
            '''
            if value < 0:
                value = 0
            if value > 100:
                value = 100
            slf._fields[name] = value
        setter(self, start)
        inst_prop = InstanceProperty(self, getter, setter)  # noqa: F841
        # Need to directly assign the instance property, or decloak it.
        setattr(self, name, getcloaked('inst_prop'))


machine = MachineState()

for letter, start in zip(['a', 'b', 'c'], [-1, 0, 1]):
    machine.add_input(letter, start)

print("Demoing instance properties:")
print()
print("This instance property only allows values between 0 and 100")
print("Instantiated with a: -1, b: 0, c: 1")
print(f"machine.a is {machine.a}")
print(f"machine.b is {machine.b}")
print(f"machine.c is {machine.c}")

# Assign a value that is too high
print("Assing a value of 200 to attribute c")
machine.c = 200

print(f"machine.c is {machine.c}")

# Template expressions                                                                                                            


class SimpleArrayExecutor:
    def __init__(self, nodes):
        if len(nodes) < 1:
            raise ValueError("There must be at least one node at"
                             " initialization")
        self.nodes = nodes
        self.length = len(nodes[0].values)
        self.cached = None

    def __getself__(self):
        if self.cached is not None:
            return self.cached
        print("Doing all the additions")
        addedValues = [0]*self.length
        for i in range(self.length):
            for node in self.nodes:
                addedValues[i] += node.values[i]

        self.cached = SimpleArray(addedValues)
        return self.cached

    def __add__(self, other):
        if not isinstance(other, SimpleArray) and not \
                isinstance(other, SimpleArrayExecutor):
            raise TypeError("Can only add SimpleArrays, or Simple"
                            " ArrayExecutors")
        if isinstance(other, SimpleArray):
            if len(other.values) != self.length:
                raise ValueError("Can only add Arrays of the same length")
            self.nodes.append(other)

        if isinstance(other, SimpleArrayExecutor):
            self.nodes += other.nodes
        return self


class SimpleArray:
    def __init__(self, iterable):
        self.values = list(iterable)

    def __add__(self, other):
        if not isinstance(other, SimpleArray) and not\
                isinstance(other, SimpleArrayExecutor):
            raise TypeError("Can only add SimpleArray to SimpleArray")
        if isinstance(other, SimpleArrayExecutor):
            return other + self
        if len(self.values) != len(other.values):
            raise ValueError("Can only add arrays of the same length")
        return SimpleArrayExecutor([self, other])


print("Creating 6 'long' arrays with 201 element (last element is 200)")
arr1 = SimpleArray(range(201))
arr2 = SimpleArray(range(201))
arr3 = SimpleArray(range(201))
arr4 = SimpleArray(range(201))
arr5 = SimpleArray(range(201))
arr6 = SimpleArray(range(201))

print("Add them all together with a single loop over the values")
print("Will only print 'Doing all the additions once")
arr7 = arr1 + arr2 + arr3 + arr4 + arr5 + arr6

print(f'The final array element of the combined array is {arr7.values[-1]}')
```

### Demo Output
```
Demoing a variable with history:

The current value of g is [1, 2, 3]
The history of g is [2, 12, 'hello world']
Rolling back the history on g
The current value of g is 12
The history of g is [2]


Demoing a variable that syncs to disk

Creating a new file backed variable, with value 'hello world'
Reassigning the value to 'Brave new world'
Close the backing file
Load back in the saved var
The file var stored the value Brave new world


Demoing an implementation of context variables:

Setting the context variable in context 1
Printing the context variable in context 1
1
Printing the context variable in context 2, it has the default value
hello world


Demoing cost variables:
The declared constant is:
100
The type of the declared constant is (i.e. the cloaking type):
<class 'int'>
The real type is:
<class '__main__.Constant'>
Attempting to reassign throws an error:
Constant variables can't be reassigned


0
0
1
Demoing instance properties:

This instance property only allows values between 0 and 100
Instantiated with a: -1, b: 0, c: 1
machine.a is 0
machine.b is 0
machine.c is 1
Assing a value of 200 to attribute c
machine.c is 100
Creating 6 'long' arrays with 201 element (last element is 200)
Add them all together with a single loop over the values
Will only print 'Doing all the additions once
Doing all the additions
The final array element of the combined array is 1200

```
