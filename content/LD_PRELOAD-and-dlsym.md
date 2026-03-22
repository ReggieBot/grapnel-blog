+++
title = "LD_PRELOAD and the dlsym recursion problem"
date = 2026-03-21
[taxonomies]
tags = ["grapnel", "c", "linux", "glibc", "ld_preload", "dlsym"]
+++

In the last post I mentioned that I was working on the `LD_PRELOAD` hook and dealing with the "dlsym recursion issue". Let's dig into that shall we? 

## Background (LD_PRELOAD & dlsym)

`LD_PRELOAD` is a Linux environment variable that tells the **dynamic linker** to load a shared library *before* all of the others, including glibc! Now, if that library defines a symbol that also exists in glibc (malloc, for example), the linker resolves to yours first, and the target process never knows! This is how grapnel intercepts allocations without ever touching the binary. Sneaky sneaky...

`dlsym` is the actual runtime symbol lookup function. When given a handle and a name, it returns a **ptr** to that symbol. The special handle `RTLD_NEXT` just means "find the next occurrence of this symbol in the load order after the current library". This is exactly how a hook is able to reach through to the real implementation of what it's wrapping. 

## The incredibly naive hook 

The most basic `malloc` interception looks something like this. This is actually what I started with

```c
#define _GNU_SOURCE
#include <dlfcn.h> 

static void *(*real_malloc)(size_t) = NULL;

void *malloc(size_t size) {
    if (!real_malloc)
        real_malloc = dlsym(RTLD_NEXT, "malloc"); // tell dynamic linker to grab next "malloc" in search order
    
    return real_malloc(size);
}

```

`RTLD_NEXT` tells the dynamic linker to find the *next* `malloc` in the library search order, this means it will skip our hook and land on the glibc's. Super clean, super obvious, and it immediately crashes. 

## Why it crashes...

After getting punched in the face by a segfault about a hundred times, I realized something. `dlsym` is not free. Internally, glibc's `dlsym` allocates memory to do its work. This means it calls `malloc`. Which our hook is intercepting. Which calls `dlsym`. Which calls `malloc`. See where I'm going with this? Our hook never actually resolves the `real_malloc`, because we're calling malloc again before we even get there. As you might've guessed, this pretty quickly outsizes the 8MB stack size. 

Now this only bites us during initialization, the very first malloc call before we've resolved the real pointer. But that's exactly when `dlsym` needs to run, so it bites *every* time. 

## The fix: a bootstrap allocator! (I think)

We need to break the cycle. The approach is two parts. 

After some serious digging, the common fix for this issue seems to be a two step approach. 

**1. A guard flag:** detect re-entrant calls into our hook: 

```c 
static __thread int in_hook = 0;
```

`__thread` makes this **per-thread** (more on why this matters in my next post!). The flag starts at `0`. We set it to `1` before calling `dlsym`, and then clear it after. Any `malloc` call that arrives while the flag is set, we now know is a re-entrant call coming from inside of `dlsym`, so we handle it differently. 

**2. A static bump allocator:** to give `dlsym` somewhere to safely allocate from.

```c

static char bootstrap_buffer[4096];
static size_t bootstrap_offset = 0; // tracking 

static void *bootstrap_alloc(size_t size) {
    // bitwise wizardry to align the allocation to a multiple of 16. I stole this. No shame.
    size = (size + 15) & ~15;
    
    if (bootstrap_offset + size > sizeof(bootstrap_buf))
        return NULL;
        
    void *ptr = bootstrap_buf + bootstrap_offset;
    
    // update our offset
    bootstrap_offset += size;
    
    return ptr;
}

```

A fixed static buffer! We are essentially just handing out pointers by bumping an offset. No `free`, since nothing allocated here needs to outlive the `dlsym` initialization.

Now, putting it all together...

```c

void *malloc(size_t size) {
    if (in_hook)
        return bootstrap_alloc(size); // straight to bump allocator, No recursion today!
        
    // If we get to here, now we can safely resolve the real 'malloc'
    in_hook = 1; 
    if (!real_malloc)
        real_malloc = dlsym(RTLD_NEXT, "malloc");
    in_hook = 0;
    
    return real_malloc(size);
}

```

When `dlsym` internally calls `malloc`, the flag (in_hook) is already set, it hits `bootstrap_alloc` and returns immediately. No further `dlsym` calls, which means, recursion broken!

## The unfortunate free() wrinkle

There's one more thing, as there always is. `dlsym` may also call `free` on pointers that it allocated during initialization. But wait, those pointers came from our bootstrap buffer, not glibc. So we need to guard `free` as well.

```c

void free(void *ptr) {
    // perform a range check to see if pointer belongs to our static buffer
    if (ptr >= (void *)bootstrap_buffer &&
    ptr < (void *)(bootstrap_buffer + sizeof(bootstrap_buffer)))
    return; // this is a bootstrap pointer, so nothing to do
    
    if (!real_free)
        real_free = dlsym(RTLD_NEXT, "free");
    real_free(ptr) // free the real allocation
}
```

This is not the most elegant extra check, but it's unfortunately necessary. Outside of parsing ELF to resolve the real symbols manually this is the only thing I can think of and it seems to be the accepted solution. That extra check will keep me up at night though. With a compiler hint and modern branch prediction, the cost is effectively zero (except my sanity). But passing a static buffer address into glibc's `free` would be much more catastrophic. 

## Where this leaves us 

The hook now survives initialization without exploding in spectacular fashion. I think that's a win. The bootstrap buffer is a one-time hack, once `real_malloc` is resolved it's never touched again. The guard flag is what's doing the real work, and right now it's a plain `__thread int`. That's good enough to ship, but the current hook is doing two jobs at once, boostrapping and production - which is a bit messy in my opinion. I'm planning to clean that up by using function pointers to separate the two phases, so the boostrap logic gets completely out of the hot path once that annoying initialization is done. I'll probably have a blog post about that sometime soon. But there's also more to say about how TLS actually works under the hood and why `__thread` is the right tool here rather than a global flag. It's actually pretty interesting and I had a great time digging into it. That's next. 

-Reggie
