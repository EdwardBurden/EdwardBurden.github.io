---
layout: post
author: Edward Burden
title : Memory Allocators.
project :  C++
category : Updates
thumbnail: stack allocator.png

excerpt: A variety of different memory allocators.
---

<style>
/* Headings: allocator names (h2) are clearly bigger than their sub-headings (h3) */
h2 { font-size: 2.25rem; font-weight: 700; margin-top: 3rem; margin-bottom: 1.25rem; padding-bottom: 0.4rem; border-bottom: 2px solid rgba(127, 127, 127, 0.35); }
h3 { font-size: 1.25rem; font-weight: 600; margin-top: 2rem; margin-bottom: 0.75rem; }
.alloc-code { background: rgba(127, 127, 127, 0.12); color: inherit; padding: 1rem; border-radius: 0.5rem; overflow-x: auto; font-size: 0.875rem; }
.alloc-code code { color: inherit; background: none; }
mark.hl { color: inherit; padding: 0.05rem 0.2rem; border-radius: 0.2rem; }
mark.hl-1 { background: rgba(0, 123, 255, 0.30); }
mark.hl-2 { background: rgba(40, 167, 69, 0.32); }
mark.hl-3 { background: rgba(255, 153, 0, 0.38); }
mark.hl-4 { background: rgba(156, 39, 176, 0.30); }
</style>

I started adding memory allocators to my engine, Gromit, when I realised how heavily it relied on the heap. In that project I made a pool allocator and a stack allocator, but here I've properly fleshed out a few custom allocators that I could actually use in a project when needed.

## The allocators

In this post I'll go through the allocators I built:

- [**Arena allocator**](#arena-allocator): the simplest one, and a good starting point.
- [**Stack allocator**](#stack-allocator): an arena that can free in reverse order.
- [**Dual stack allocator**](#dual-stack-allocator): one block of memory with a stack growing from each end.
- [**Linked stack allocator**](#linked-stack-allocator): a stack where you release memory using the object itself.
- [**Pool allocator**](#pool-allocator): a pool of same-sized elements.

## Arena allocator {#arena-allocator}

<div class="row align-items-center mb-5">
<div class="col-md-6" markdown="1">

```cpp
class ArenaAllocator : public Allocator
{
public:
    ArenaAllocator(const size_t size);
    ArenaAllocator(const ArenaAllocator& other) = delete;
    ArenaAllocator& operator=(const ArenaAllocator& other) = delete;
    ArenaAllocator(ArenaAllocator&& other) = delete;
    ArenaAllocator& operator=(ArenaAllocator&& other) = delete;
    ~ArenaAllocator();

    void* Allocate(const size_t size, const size_t alignment = alignof(std::max_align_t));
    void Reset() override;

private:
    std::byte* m_marker;
};
```

</div>
<div class="col-md-6" markdown="1">

As you can see from the header file for the arena allocator, I made the simplest possible version. You can allocate up to its limit, and you can reset the whole thing, which "frees" the memory. The memory is only actually freed when the allocator is destroyed.

This is useful for storing resources that are used throughout the lifetime of the application, which you still want to manage manually without pushing them onto the heap.

</div>
</div>

## Stack allocator {#stack-allocator}

<div class="row align-items-center mb-4">
<div class="col-md-5 mb-3 mb-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/stack%20allocator.png" alt="Diagram of the stack allocator">
</div>
<div class="col-md-7" markdown="1">

This image is from Jason Gregory's *Game Engine Architecture*, and it shows how the stack allocator works. It's an improvement on the arena, because you can now "unload" some memory by freeing down to a point in the allocated stack. That point has to be a valid position in the stack.

The code below shows how allocation is handled. There's a bit going on, so let's walk through it.

</div>
</div>

<pre class="alloc-code"><code>void* StackAllocator::Allocate(const size_t size, const size_t alignment)
{
    if (!AllocatorUtils::AddressIsPowerOf2(alignment))
        return nullptr;

    std::byte* alignedMarker = nullptr;
    {
        std::lock_guard&lt;std::mutex&gt; lock(m_mutex);
        <mark class="hl hl-1">alignedMarker = m_marker + (alignment - 1);</mark>
        <mark class="hl hl-1">AllocatorUtils::AlignPointer(alignedMarker, alignment);</mark>
        <mark class="hl hl-2">if (!AllocatorUtils::CheckMemoryBounds(alignedMarker + size, m_bytes, m_limit))</mark>
            <mark class="hl hl-2">return nullptr;</mark>

#ifndef NDEBUG
        std::memset(m_marker, 'P', alignedMarker - m_marker);
        std::memset(alignedMarker, 'A', size);
#endif //NDEBUG

        <mark class="hl hl-2">m_marker = alignedMarker + size;</mark>
    }
    <mark class="hl hl-3">return static_cast&lt;void*&gt;(alignedMarker);</mark>
}</code></pre>

The first thing to know is that all of my allocators add padding, so that the requested memory starts at an address that is correctly aligned, meaning it is divisible by the requested alignment. There are two reasons for this:

- **Speed:** a CPU can read aligned data in a single access, whereas misaligned data can span two, which makes it slower.
- **Requirements:** some architectures expect all memory requests to be aligned.

This <mark class="hl hl-1">padding calculation</mark> is handled in a utility class that all of the allocators have access to. With the correct padding applied, <mark class="hl hl-1">the aligned marker is calculated</mark>, and this is where the actual allocation can begin.

Next I do a <mark class="hl hl-2">quick bounds check</mark> to make sure the allocation, now including the extra padding, isn't too large for the allocator's memory. If it fits, <mark class="hl hl-2">the allocator's marker is moved forward</mark>.

What's important is what I send back to the caller. I don't return the allocator's marker. <mark class="hl hl-3">I return the aligned marker from before the size was added</mark>, which is the start of the allocation, as a `void*`. The caller uses that pointer for the object they want to create, and nothing else. It is not the pointer to use when releasing memory.

To release memory, you first get the allocator's marker with `GetMarker()`, which returns it as a `std::byte*`. Store that marker and allocate as normal. When you're finished, call `FreeToMarker` with the stored marker, and everything allocated after it is released.

On the left is the templated `Allocate`, which fills in the size and alignment of the type for you, along with `GetMarker`. On the right is an example of using the stack allocator.

<div class="row mb-5">
<div class="col-md-6 mb-3 mb-md-0" markdown="1">

```cpp
template<typename T>
void* Allocate()
{
    return Allocate(sizeof(T), alignof(T));
}

inline std::byte* GetMarker() const { return m_marker; };
```

</div>
<div class="col-md-6" markdown="1">

```cpp
StackAllocator* stack = new StackAllocator(256);
auto marker = stack->GetMarker();
float* floatExample = static_cast<float*> (stack->Allocate<float>());
auto classExample = stack->Allocate<ExampleClass>();
stack->FreeToMarker(marker);
```

</div>
</div>

<div class="row align-items-center mb-5">
<div class="col-md-6 order-md-2" markdown="1">

## Dual stack allocator {#dual-stack-allocator}

The dual stack allocator works really the same as the regular stack allocator. The difference is that you can now allocate and free up, and also allocate and free down. That means there are two markers, one to manage each direction.

The extra checks it needs are there to make sure the two areas of memory never overlap.

</div>
<div class="col-md-6 order-md-1 mt-3 mt-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/doublended.png" alt="Diagram of the dual stack allocator">
</div>
</div>

A `StackArea` enum chooses which end to use, and the two markers start at opposite ends of the same block of memory:

<div class="row mb-4">
<div class="col-md-6 mb-3 mb-md-0" markdown="1">

```cpp
enum class StackArea
{
    Top = 0,
    Bottom = 1
};
```

</div>
<div class="col-md-6" markdown="1">

```cpp
m_topMarker = m_bytes + m_size;
m_bottomMarker = m_bytes;
```

</div>
</div>

### Why the marker is required

When I first made the stack allocator, it annoyed me that you had to cache the marker before allocating. I wanted to allocate, get the object, and then use that to free back.

But when I built the top of the dual stack, the marker made total sense. Allocating from the top is the normal stack upside down: the address you get back is the start of the object, but the point you need to free back to is the far end of it. The returned address can't tell you where to free back to, so you keep the marker.


## Linked stack allocator {#linked-stack-allocator}

The linked stack allocator is one of my favourites to use, and it was a lot of fun to make.

The premise is that you can allocate some memory, either by specifying a number of bytes or by using the template. The object (a `void*`) you get back from the allocation is then what you pass back in as the marker, so you no longer need to cache the marker before allocating.

Another great feature is that it validates whether the marker you pass in is a piece of memory that was actually allocated. With the normal stack allocator, as long as the marker was within the bounds it would be freed. Now you can only free previously allocated markers.

### How it works

So how is this done?

Each time you ask the linked stack allocator for some memory, it creates a small header in memory:

```cpp
struct Header
{
    uint32_t offset; // The offset from the current marker to the last marker.
    uint8_t padding; // The padding added before the actual allocation.
};
```

### Seeing it in memory

In debug builds the allocator fills memory with recognisable patterns: `H` for headers, `A` for allocated memory, `P` for padding and `U` for unused memory. A header is 8 bytes: the 4 byte `offset`, the 1 byte `padding`, and 3 bytes of struct padding that are never written to, so they stay as `H`.
Here the first allocation is complete, with its header (offset 0, padding 8) followed by five `A` bytes. The second allocation is caught part way through: its padding and header areas are filled in, but the header's values haven't been written yet.

<img class="img-fluid rounded shadow mb-3" src="/assets/images/link.png" alt="Debugger memory view showing the first allocation complete and the second allocation part way through">

Here is the same area of memeory after a few more allocations, you can see the alignment through paddign stays consistent to keep the allocations aligned.

<img class="img-fluid rounded shadow mb-3" src="/assets/images/linked2.png" alt="Debugger memory view showing four allocations, each with a header, allocated memory and padding">

### Freeing memory

This is the other half of the linked stack allocator. `FreeMarker` takes the pointer you were given when you allocated, and uses it as the marker.

<pre class="alloc-code"><code>void LinkedStackAllocator::FreeMarker(void* ptr)
{
    <mark class="hl hl-1">if (ptr == nullptr)</mark>
        <mark class="hl hl-1">return;</mark>

    std::lock_guard&lt;std::mutex&gt; lock(m_mutex);

    <mark class="hl hl-1">if (!AllocatorUtils::CheckMemoryBounds(ptr, m_bytes, m_marker))</mark>
        <mark class="hl hl-1">return;</mark>

    <mark class="hl hl-1">if (IsEmpty())</mark>
        <mark class="hl hl-1">return;</mark>

    std::byte* marker = static_cast&lt;std::byte*&gt;(ptr);
    <mark class="hl hl-2">Header* header = m_header;</mark>
    while (true)
    {
        <mark class="hl hl-2">std::byte* headerByte = reinterpret_cast&lt;std::byte*&gt;(header);</mark>
        <mark class="hl hl-2">std::byte* markerByte = headerByte + sizeof(Header);</mark>
        <mark class="hl hl-3">if (markerByte == marker)</mark>
        {
            <mark class="hl hl-3">FreeHeader(header);</mark>
            <mark class="hl hl-3">return;</mark>
        }
        <mark class="hl hl-4">if (header-&gt;offset == 0)</mark>
            <mark class="hl hl-4">break;</mark>

        <mark class="hl hl-2">header = reinterpret_cast&lt;Header*&gt;(headerByte - header-&gt;offset);</mark>
    }
}</code></pre>

It starts with some <mark class="hl hl-1">quick checks</mark>. A null pointer is ignored, the pointer has to be inside the memory that has been allocated so far (between the start of the block and the current marker), and an empty allocator has nothing to free.

Then comes the validation. <mark class="hl hl-2">Starting at the most recent header, the allocator walks back through the chain.</mark> For each header it <mark class="hl hl-2">works out the address just after it</mark>, which is where that header's object starts, and <mark class="hl hl-3">checks whether that matches the pointer you passed in</mark>. If it does, the pointer is a real allocation, and <mark class="hl hl-3"><code>FreeHeader</code> rolls the allocator back</mark> to that point. If <mark class="hl hl-4">a header's offset is 0</mark>, the walk has reached the first header without finding a match, so the pointer wasn't a valid allocation and nothing is freed.

This walk is the cost of the validation. Freeing the most recent allocation is quick, but freeing an older one means stepping back through every header after it. And while this means you can pass in any valid object that was allocated, you still need to free them in the correct order.

<div class="row align-items-center mb-5">
<div class="col-md-6 order-md-2" markdown="1">

## Pool allocator {#pool-allocator}

The pool allocator is very similar to the one I made when working on Gromit, but with a specific improvement.

Like the others, it gets one big block of contiguous memory, but then it portions that block out into elements of a fixed size. That means a pool allocator can only hand out one consistent size. It's best used when you want lots of the same type of object, such as making as many vectors as you want without worrying about the heap. For example:

</div>
<div class="col-md-6 order-md-1 mt-3 mt-md-0">

<img class="img-fluid rounded shadow" src="/assets/images/Allocator1.png" alt="Diagram of the pool allocator">

</div>
</div>

```cpp
PoolAllocator<vector3>* vectorPool = new PoolAllocator<vector3>(2);
auto v1 = vectorPool->Allocate(1, 4, 1);
auto v2 = vectorPool->Allocate(20, 75.5, 2);
```

But it's up to you how you use it.

### How it works

My pool allocator multiplies the element size by the number of elements, and requests that much memory up front. It then loops through that block, starting from the base. At the start of each element, it stores a pointer to the next element.

This gives me a linked list of pointers to the next available element, and the list lives inside the free elements themselves. That means I don't need a separate list stored somewhere, and I don't have to add a pointer to every element, which would increase the total size.

The caveat is that every element must be at least `sizeof(void*)` for this to work. For what I'll actually use this pool allocator for, I think that's okay.

When you request an element, finding one is easy. The allocator <mark class="hl hl-1">takes the first available element</mark>, checks that it links to a next available element, then <mark class="hl hl-2">reads the pointer stored inside it and moves its own pointer over to that next element</mark>, and returns the element you asked for.

<pre class="alloc-code"><code><mark class="hl hl-1">allocation = reinterpret_cast&lt;std::byte*&gt;(m_freeElement);</mark>
<mark class="hl hl-2">std::byte** elementPointer = reinterpret_cast&lt;std::byte**&gt;(m_freeElement);</mark>
<mark class="hl hl-2">m_freeElement = reinterpret_cast&lt;T*&gt;(*elementPointer);</mark></code></pre>

When freeing, you pass back the element you were given when you allocated it, just like the linked stack allocator. The destructor is called on it to make sure it's actually unused. Then the pointer is <mark class="hl hl-3">cast to a pointer to a byte pointer (<code>std::byte**</code>), so that whatever is stored there can be replaced with a pointer to the currently free element</mark>. Finally, <mark class="hl hl-4">the pointer that was passed in becomes the next available element</mark>, which puts the freed element at the front of the list.

<pre class="alloc-code"><code>std::lock_guard&lt;std::mutex&gt; lock(m_mutex);
<mark class="hl hl-3">std::byte** elementPointer = reinterpret_cast&lt;std::byte**&gt;(ptr);</mark>
<mark class="hl hl-3">*elementPointer = reinterpret_cast&lt;std::byte*&gt;(m_freeElement);</mark>
<mark class="hl hl-4">m_freeElement = ptr;</mark></code></pre>

## Comparing them

Here is how the five allocators compare:

<div class="table-responsive mb-4">
<table class="table table-bordered">
<thead>
<tr><th>Allocator</th><th>Sizes</th><th>Frees</th><th>Speed</th><th>Best for</th></tr>
</thead>
<tbody>
<tr><th scope="row">Arena</th><td>Any size, up to the limit</td><td>Everything at once, with <code>Reset</code></td><td>Fast, it just moves a marker</td><td>Data that lives for the whole application</td></tr>
<tr><th scope="row">Stack</th><td>Any size, up to the limit</td><td>Back to a saved marker, in reverse order</td><td>Fast, it just moves a marker</td><td>Temporary data that is released in groups</td></tr>
<tr><th scope="row">Dual stack</th><td>Any size, from either end</td><td>Back to a saved marker, for each end</td><td>Fast, with an extra check that the two ends don't overlap</td><td>Two groups of data sharing one block of memory</td></tr>
<tr><th scope="row">Linked stack</th><td>Any size, up to the limit</td><td>By passing back the object itself, in the correct order</td><td>Allocating is fast, but freeing walks back through the headers, so older allocations cost more</td><td>Stack-style allocation without having to save markers</td></tr>
<tr><th scope="row">Pool</th><td>One fixed size</td><td>Any single element, in any order</td><td>Fast, allocating and freeing take the same time every time</td><td>Lots of objects of the same type</td></tr>
</tbody>
</table>
</div>

The right choice depends on how the memory will be used. If everything is released together, the arena is enough. If it is released in groups, in reverse order, a stack does the job, and the linked stack lets you free by object instead of by marker. If you need lots of objects of the same type that come and go in any order, the pool is the one to use.