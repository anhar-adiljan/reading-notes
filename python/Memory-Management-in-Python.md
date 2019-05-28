# Reading Note: Memory Management in Python

Original blog post link:

<https://realpython.com/python-memory-management/>

## Memory is an empty book - an analogy

- Authors: applications and processes that need to store data in memory
- Manager: memory manager (decides where the "authors" can write in a book)
- Cleaner: garbage collector (removes old "stories" to make room for new ones)

## Memory management: from hardware to software

- Two aspects of memory management: allocation & freeing
- Layers of abstraction above the hardware: OS, Python implementation

## Python Implementation

- What does a Python implementation do?
  - Interpret written Python code
  - Execute interpreted code on a computer
- Default Python implementation: CPython (written in C)
  - `struct PyObject`: used by every object in CPython
    - `ob_refcount`: reference count used for garbage collection
    - `ob_type`: pointer to the actual object type, i.e. another struct that
    describes the Python object
  - Each object has its object-specific memory allocator and deallocator.
- Other implementations
  - IronPython: for Microsoft's Common Language runtime
  - Jython: for Java Virtual Machine (JVM)
  - PyPy

## The Global Interpreter Lock (GIL)

- Lock the entire interpreter to ensure that only a single thread interacts
with the shared resource (e.g. memory)
- Heavily debated in the Python community
- Reference article: "What is the Python Global Interpreter Lock (GIL)?"

## Garbage Collection

- "Free" the memory allocated for an object when its reference count drops to 0.
- Note on reference count
  - Ways to increase reference count
    - Assign the object to other variable
    - Pass the object as an argument
    - Include the object in a list etc.
  - To inspect reference count of an object
    - `sys.getrefcount(obj)`
    - Side effect: increase the reference count by 1

## CPython Memory Management

- Virtual memory for Python process includes:
  - Object-specific memory
  - Python core non-object memory
- Three main pieces: arena, pool, block

### Arena

- Aligned on a page boundary in memory
  - Python's assumption: system page size = 256KB
  - Consists of pools
- Organized as a double-linked list called `usable_arenas`
  - Sorted by the number of free pools available
  - The arena that is the most full of data will be selected first to place
  new data into.
- Notion of truly freeing memory
  - Truly freeing memory means returning allocated memory to OS.
  - This is only done at the arena level.

### Pool

- Placed within an arena
  - Size: 4KB (size of virtual memory page)
  - Composed of blocks from a single size class
  - Maintains a singly-linked list of "free" blocks of memory (`freeblock`),
  with new blocks added to the front
- Organized as doubly-linked lists grouped by its size class
- 3 states of existence
  - `used`: has available block
  - `full`: fully allocated
  - `empty`: has no data and can be assigned any size class
- 2 lists
  - `usedpools`: tracks all pools that have some space available for data
  of each size class
  - `freepools`: tracks all empty pools

### Block

- 3 states
  - `untouched`: not allocated
  - `free`: allocated but later freed
  - `allocated`: contains relevant data
- Not enough free blocks: then allocate untouched blocks in the pool

### Python Memory Allocator

- "Strives at all levels (arena, pool, and block) never to touch a piece of
memory until it's actually needed."
  - Stated in the comments of CPython implementation
  - Purpose: to reduce the overall memory footprint
- Size class

| Request in Bytes | Size of Allocated Block | Size Class Index |
|:----------------:|:-----------------------:|:----------------:|
|        1-8       |            8            |         0        |
|       9-16       |            16           |         1        |
|        ...       |           ...           |        ...       |
|      505-512     |           512           |        63        |