Python Memory Management
========================

One of the major challenges in writing (somewhat) large-scale Python
programs is to keep memory usage at a minimum. However, managing memory in
Python is easy—if you just don't care. Python allocates memory
transparently, manages objects using a reference count system, and frees
memory when an object's reference count falls to zero. In theory, it's
swell. In practice, you need to know a few things about Python memory
management to get a memory-efficient program running. One of the things you
should know, or at least get a good feel about, is the sizes of basic
Python objects. Another thing is how Python manages its memory internally.

So let us begin with the size of basic objects. In Python, there's not a
lot of primitive data types: there are ints, longs (an unlimited
precision version of ints), floats (which are doubles), tuples, strings,
lists, dictionaries, and classes.

Basic Objects
-------------

What is the size of ``int``? A programmer with a C or C++ background will
probably guess that the size of a machine-specific ``int`` is something
like 32 bits, maybe 64; and that therefore it occupies at most 8 bytes. But
is that so in Python?

Let us first write a function that shows the sizes of objects (recursively
if necessary):

::

    import sys

    def show_sizeof(x, level=0):

        print "\t" * level, x.__class__, sys.getsizeof(x), x

        if hasattr(x, '__iter__'):
            if hasattr(x, 'items'):
                for xx in x.items():
                    show_sizeof(xx, level + 1)
            else:
                for xx in x:
                    show_sizeof(xx, level + 1)

We can now use the function to inspect the sizes of the different basic
data types:

::

        show_sizeof(None)
        show_sizeof(3)
        show_sizeof(2**63)
        show_sizeof(102947298469128649161972364837164)
        show_sizeof(918659326943756134897561304875610348756384756193485761304875613948576297485698417)

If you have a 32-bit 2.7x Python, you'll see:

::

      8 None
      12 3
      22 9223372036854775808
      28 102947298469128649161972364837164
      48 918659326943756134897561304875610348756384756193485761304875613948576297485698417

and if you have a 64-bit 2.7x Python, you'll see:

::

      16 None
      24 3
      36 9223372036854775808
      40 102947298469128649161972364837164
      60 918659326943756134897561304875610348756384756193485761304875613948576297485698417

Let us focus on the 64-bit version (mainly because that's what we need the
most often in our case). ``None`` takes 16 bytes. ``int`` takes 24 bytes,
*three times* as much memory as a C ``int64_t``, despite being some kind of
"machine-friendly" integer. Long integers (unbounded precision), used to
represent integers larger than 2\ :sup:`63`\ -1, have a minimum size of 36
bytes. Then it grows linearly in the logarithm of the integer represented.

Python's floats are implementation-specific but seem to be C doubles.
However, they do not eat up only 8 bytes:

::

       show_sizeof(3.14159265358979323846264338327950288)

Outputs

::

      16 3.14159265359

on a 32-bit platform and

::

      24 3.14159265359

on a 64-bit platform. That's again, three times the size a C programmer
would expect. Now, what about strings?

::

        show_sizeof("")
        show_sizeof("My hovercraft is full of eels")

outputs, on a 32 bit platform:

::

      21
      50 My hovercraft is full of eels

and

::

      37
      66 My hovercraft is full of eels

An *empty* string costs 37 bytes in a 64-bit environment! Memory used
by string then linearly grows in the length of the (useful) string.

\*
\* \*

Other structures commonly used, tuples, lists, and dictionaries are
worthwhile to examine. Lists (which are implemented as `array
lists <http://en.wikipedia.org/wiki/Dynamic_array>`_, not as `linked
lists <http://en.wikipedia.org/wiki/Linked_list>`_, with `everything it
entails <http://en.wikipedia.org/wiki/Dynamic_array#Performance>`_) are
arrays of references to Python objects, allowing them to be
heterogeneous. Let us look at our sizes:

::

        show_sizeof([])
        show_sizeof([4, "toaster", 230.1])

outputs

::

      32 []
      44 [4, 'toaster', 230.1]

on a 32-bit platform and

::

      72 []
      96 [4, 'toaster', 230.1]

on a 64-bit platform. An empty list eats up 72 bytes. The size of an
empty, 64-bit C++ ``std::list()``is only 16 bytes, 4-5 times less. What
about tuples? (and dictionaries?):

::

        show_sizeof({})
        show_sizeof({'a':213, 'b':2131})

outputs, on a 32-bit box

::

     136 {}
      136 {'a': 213, 'b': 2131}
             32 ('a', 213)
                     22 a
                     12 213
             32 ('b', 2131)
                     22 b
                     12 2131

and

::

     280 {}
      280 {'a': 213, 'b': 2131}
             72 ('a', 213)
                     38 a
                     24 213
             72 ('b', 2131)
                     38 b
                     24 2131

for a 64-bit box.

This last example is particularly interesting because it "doesn't add up."
If we look at individual key/value pairs, they take 72 bytes (while their components
take 38+24=62 bytes, leaving 10 bytes for the pair itself), but the
dictionary takes 280 bytes (rather than a strict minimum of 144=72×2
bytes). The dictionary is supposed to be an efficient data structure for
search and the two likely implementations will use more space that strictly
necessary. If it's some kind of tree, then we should pay the cost of
internal nodes that contain a key and two pointers to children nodes; if
it's a hash table, then we must have some room with free entries to ensure
good performance.

The (somewhat) equivalent ``std::map`` C++ structure takes 48 bytes when
created (that is, empty). An empty C++ string takes 8 bytes (then allocated
size grows linearly the size of the string). An integer takes 4 bytes (32 bits).

\*
\* \*

Why does all this matter? It seems that whether an empty string takes 8
bytes or 37 doesn't change anything much. That's true. That's true *until*
you need to scale. Then, you need to be really careful about how many
objects you create to limit the quantity of memory your program uses. It is
a problem in real-life applications. However, to devise a really good
strategy about memory management, we must not only consider the sizes of
objects, but how many and in which order they are created. It turns out to
be very important for Python programs. One key element to understand is how
Python allocates its memory internally, which we will discuss next.

Internal Memory Management
--------------------------

To speed-up memory allocation (and reuse) Python uses a number of lists
for small objects. Each list will contain objects of similar size: there
will be a list for objects 1 to 8 bytes in size, one for 9 to 16, etc.
When a small object needs to be created, either we reuse a free block in
the list, or we allocate a new one.

There are some internal details on how Python manages those lists into
blocks, pools, and "arena": a number of block forms a pool, pools are
gathered into arena, etc., but they're not very relevant to the point we
want to make (if you really want to know, read Evan Jones' `ideas on how to
improve Python's memory allocation
<http://www.evanjones.ca/memoryallocator/>`_). The important point is that
those lists *never shrink*.

Indeed: if an item (of size *x*) is deallocated (freed by lack of
reference) its location is not returned to Python's global memory pool (and
even less to the system), but merely marked as free and added to the free
list of items of size *x*. The dead object's location will be reused if
another object of compatible size is needed. If there are no dead objects
available, new ones are created.

If small objects memory is never freed, then the inescapable conclusion is
that, like goldfishes, these small object lists only keep growing, never
shrinking, and that the memory footprint of your application is dominated
by the largest number of small objects allocated at any given point.

\*
\* \*

Therefore, one should work hard to allocate only the number of small
objects necessary for one task, favoring (otherwise *unpythonèsque*) loops
where only a small number of elements are created/processed rather than
(more *pythonèsque*) patterns where lists are created using list generation
syntax then processed.

While the second pattern is more *à la Python*, it is rather the worst
case: you end up creating lots of small objects that will come populate the
small object lists, and even once the list is dead, the dead objects (now
all in the free lists) will still occupy a lot of memory.

\*
\* \*

The fact that the free lists grow does not seem like much of a problem
because the memory it contains is still accessible to the Python program.
But from the OS's perspective, your program's size is the total (maximum)
memory allocated to Python. Since Python returns memory to the OS on the
heap (that allocates other objects than small objects) only on Windows, if
you run on Linux, you can only see the total memory used by your program
increase.

\*
\* \*

Let us prove my point using `memory\_profiler
<http://pypi.python.org/pypi/memory_profiler>`_, a Python add-on module
(which depends on the ``python-psutil`` package) by `Fabian Pedregosa
<https://github.com/fabianp>`_ (the module's `github page
<https://github.com/fabianp/memory_profiler>`_). This add-on provides the
decorator ``@profile`` that allows one to monitor one specific function
memory usage. It is extremely simple to use. Let us consider this small
program (it makes my point entirely):

::

    import copy
    import memory_profiler

    @profile
    def function():
        x = range(1000000)  # allocate a big list
        y = copy.deepcopy(x)
        del x
        return y

    if __name__=="__main__":
        function()

invoking

::

    python -m memory_profiler memory-profile-me.py

prints, on a 64-bit computer

::

    Filename: memory-profile-me.py

    Line #    Mem usage    Increment   Line Contents
    ================================================
         3                             @profile
         4      9.11 MB      0.00 MB   def function():
         5     40.05 MB     30.94 MB       x=range(1000000) # allocate a big list
         6     89.73 MB     49.68 MB       y=copy.deepcopy(x)
         7     82.10 MB     -7.63 MB       del x
         8     82.10 MB      0.00 MB       return y

This small program creates a list with 1,000,000 ints (at 24 bytes each,
for ~24 million bytes) plus a list of references (at 8 bytes each, for ~8
million bytes), for about 30MB. It then deep-copies the object (which
allocates ~50MB, not sure why; a simple copy would allocate only 8MB of
references, plus about 24MB for the objects themselves---so there's a large
overhead here, maybe Python grew its heap preemptively). Freeing ``x`` with
``del`` frees the reference list, kills the associated objects, but lo!,
the amount of memory only goes down by the number of references, because
the list itself is not in a small objects' list, but on the heap, and the
dead small objects remain in the free list, and not returned to the
interpreter's global heap.

In this example, we end up with *twice* the memory allocated, with 82MB,
while only one list necessitating about 30MB is returned. You can see why
it is easy to have memory just increase more or less surprisingly if we're
not careful.


Pickle
------

On a related note: is ``pickle`` wasteful?

`Pickle <http://docs.python.org/library/pickle.html>`_ is the standard way
of (de)serializing Python objects to file. What is its memory footprint?
Does it create extra copies of the data or is it rather smart about it?
Consider this short example:

::

    import memory_profiler
    import pickle
    import random

    def random_string():
        return "".join([chr(64 + random.randint(0, 25)) for _ in xrange(20)])

    @profile
    def create_file():
        x = [(random.random(),
              random_string(),
              random.randint(0, 2 ** 64))
             for _ in xrange(1000000)]

        pickle.dump(x, open('machin.pkl', 'w'))

    @profile
    def load_file():
        y = pickle.load(open('machin.pkl', 'r'))
        return y

    if __name__=="__main__":
        create_file()
        #load_file()

With one invocation to profile the creation of the pickled data, and one
invocation to re-read it (you comment out the function not to be
called). Using ``memory_profiler``, the creation uses a lot of memory:

::

    Filename: test-pickle.py

    Line #    Mem usage    Increment   Line Contents
    ================================================
         8                             @profile
         9      9.18 MB      0.00 MB   def create_file():
        10      9.33 MB      0.15 MB       x=[ (random.random(),
        11                                      random_string(),
        12                                      random.randint(0,2**64))
        13    246.11 MB    236.77 MB           for _ in xrange(1000000) ]
        14
        15    481.64 MB    235.54 MB       pickle.dump(x,open('machin.pkl','w'))

and re-reading a bit less:

::

    Filename: test-pickle.py

    Line #    Mem usage    Increment   Line Contents
    ================================================
        18                             @profile
        19      9.18 MB      0.00 MB   def load_file():
        20    311.02 MB    301.83 MB       y=pickle.load(open('machin.pkl','r'))
        21    311.02 MB      0.00 MB       return y

So somehow, *pickling* is very bad for memory consumption. The initial list
takes up more or less 230MB, but pickling it creates an extra 230-something
MB worth of memory allocation.

Unpickling, on the other hand, seems fairly efficient. It does create more
memory than the original list (300MB instead of 230-something) but it does
not double the quantity of allocated memory.

Overall, then, (un)pickling should be avoided for memory-sensitive
applications. What are the alternatives? Pickling preserves all the
structure of a data structure, so you can recover it exactly from the
pickled file at a later time. However, that might not always be needed. If
the file is to contain a list as in the example above, then maybe a simple
flat, text-based, file format is in order. Let us see what it gives.

A naïve implementation would give:

::

    import memory_profiler
    import random
    import pickle

    def random_string():
        return "".join([chr(64 + random.randint(0, 25)) for _ in xrange(20)])

    @profile
    def create_file():
        x = [(random.random(),
              random_string(),
              random.randint(0, 2 ** 64))
             for _ in xrange(1000000) ]

        f = open('machin.flat', 'w')
        for xx in x:
            print >>f, xx
        f.close()

    @profile
    def load_file():
        y = []
        f = open('machin.flat', 'r')
        for line in f:
            y.append(eval(line))
        f.close()
        return y

    if __name__== "__main__":
        create_file()
        #load_file()

Creating the file:

::

    Filename: test-flat.py

    Line #    Mem usage    Increment   Line Contents
    ================================================
         8                             @profile
         9      9.19 MB      0.00 MB   def create_file():
        10      9.34 MB      0.15 MB       x=[ (random.random(),
        11                                      random_string(),
        12                                      random.randint(0, 2**64))
        13    246.09 MB    236.75 MB           for _ in xrange(1000000) ]
        14
        15    246.09 MB      0.00 MB       f=open('machin.flat', 'w')
        16    308.27 MB     62.18 MB       for xx in x:
        17                                     print >>f, xx

and reading the file back:

::

    Filename: test-flat.py

    Line #    Mem usage    Increment   Line Contents
    ================================================
        20                             @profile
        21      9.19 MB      0.00 MB   def load_file():
        22      9.34 MB      0.15 MB       y=[]
        23      9.34 MB      0.00 MB       f=open('machin.flat', 'r')
        24    300.99 MB    291.66 MB       for line in f:
        25    300.99 MB      0.00 MB           y.append(eval(line))
        26    301.00 MB      0.00 MB       return y

Memory consumption on writing is now much better. It still creates a lot of
temporary small objects (for 60MB's worth), but it's not doubling memory
usage. Reading is comparable (using only marginally less memory).

This particular example is trivial but it generalizes to strategies where
you don't load the whole thing first then process it but rather read a few
items, process them, and reuse the allocated memory. Loading data to a
Numpy array, for example, one could first create the Numpy array, then read
the file line by line to fill the array: this allocates one copy of the
whole data. Using pickle, you would allocate the whole data (at least)
twice: once by pickle, and once through Numpy.

Or even better yet: use Numpy (or PyTables) arrays. But that's a different
topic. In the mean time, you can have a look at `loading and saving
<http://deeplearning.net/software/theano/tutorial/loading_and_saving.html>`_
another tutorial in the Theano/doc/tutorial directory.

\*
\* \*

Python design goals are radically different than, say, C design goals.
While the latter is designed to give you good control on what you're doing
at the expense of more complex and explicit programming, the former is
designed to let you code rapidly while hiding most (if not all) of the
underlying implementation details. While this sounds nice, in a production
environment ignoring the implementation inefficiencies of a language can
bite you hard, and sometimes when it's too late. I think that having a good
feel of how inefficient Python is with memory management (by design!) will
play an important role in whether or not your code meets production
requirements, scales well, or, on the contrary, will be a burning hell of
memory.
