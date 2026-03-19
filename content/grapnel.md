+++
title = "grapnel: Building a heap fragmentation visualizer in C"
date = 2026-03-19
[taxonomies]
tags = ["grapnel", "c", "linux", "glibc", "tui"]
+++

A couple weeks ago I started building a tool I couldn't find anywhere else: a real-time heap fragmentation visualizer that runs in the terminal. Real-time, lightweight, and visually intuitive. It's called 'grapnel'. This is the first in a series of posts where I write about the design decisions, pain-points, and overall gripes as I build it. 

## What actually is external fragmentation of the heap?
When a program allocates and frees memory repeatedly, the heap develops these "gaps"; freed regions scattered between live allocations. The problem is that the allocator can't always reuse these gaps efficiently. So even if there's *technically* enough free memory, a large allocation can still fail due to there not being a large enough contiguous block. That's external fragmentation.

## How it works 
**This is still subject to change**
grapnel hooks into your program's allocator at **runtime** using `LD_PRELOAD`, intercepting every single `malloc`, `free`, and `realloc` call without modifying the target binary. These events are then passed over a lock-free ring buffer via shared memory (`mmap` + `MAP_SHARED`) to a separate visualizer process, which maps them onto a 2D spatial layout using a *Hilbert curve* and renders the result as a TUI using `notcurses`.

The architecture will look something like this:
```

  Target Process                        grapnel visualizer                        
------------------                    ---------------------                       
   malloc() call                                                                  
                                                      
         |                                                                        
         |                                                                        
         |                                                                        
         v                                                                        
   LD_PRELOAD hook                                                                
         |                mmap                                                    
         +--------------------------->   lock-free ring buffer                    
                                                   |                              
                                                   |                              
                                                   |                              
                                                   v                              
                                             red-black tree                       
                                          (allocation tracker)                    
                                                   |                              
                                                   |                              
                                                   v                              
                                      Hilbert + Quadtree mapping
                                                   |                              
                                                   |                              
                                                   v                              
                                            notcurses TUI                         
                                            
``` 

## Why these design choices?
**LD_PRELOAD over a custom allocator:** I wanted grapnel to work on existing binaries with zero recompilation from the end user. LD_PRELOAD lets me intercept allocator calls at the dynamic linker level. Common memory profilers take this approach already, *Heaptrack* for example (https://github.com/kde/heaptrack).

**Lock-free ring buffer for IPC:** The ring buffer itself was a pretty obvious choice for this kind of event stream. Ring buffers thrive in single-producer single-consumer (SPSC) setups such as this one. the more interesting decision I believe was the synchronization strategy. The hook runs inside the target process on potentially extremely hot allocation paths, so it needs to write events and get out as quick as possible to reduce the risk of slowing down the target process. A *mutex-protected* buffer would mean the hook could block waiting for the visualizer to release the lock, adding latency directly to the target process allocations - something I'm desperately trying to avoid due the to ethos of this project (lightweight, fast, invisible). Using atomic operations instead means the hook never blocks, it just writes the events and moves on.

**Hilbert curve + Quadtree for spatial mapping:** The crux of my program. A linear layout of heap addresses would just be a flat bar, thus completely destroying the spatial locality of the heap. The Hilbert curve maps 1D address space into 2D while preserving that locality. Addresses that are close in memory stay close on screen. This makes fragmentation patterns actually visible as shapes. I'm also planning on implementing a Quadtree structure to allow users to 'drill down' to increasingly granular sections of the heap. The Hilbert curve and Quadtree play very nice with each other due to both using a power-of-2 structure. 

**Red-black tree for allocation:** The obvious alternative (and my first instinct) was a lock-free hash table, which would give near instant average-case lookups on the hot path of matching a `free` to its original `malloc`. But the red-black tree is doing a lot more than just tracking live allocations. I need to normalize the heap addresses into a virtual contiguous space for the Hilbert curve mapping to work well, and to compute fragmentation statistics like free/live rations across address ranges. Both of those operations need ordered traversal, which a hash table can't give me - I'd be walking the entire table every time. The red-black tree keeps everything sorted by address, so range queries and in-order traversal come for free alongside the 'fast-enough' O(log n) insert and delete. 

**Where it's at:** Pre-alpha. I have just started fleshing out the skeleton for the LD_PRELOAD hook. This is *the* hot path, so I'm trying to make the hook as clean and performant as possible. Currently researching thread-local-storage and static buffers for solving the dlsym recursion issue. 

I'm building this deliberately. Understanding every line before moving on to the next. These posts are sort of my way for thinking out loud as I go.

Next up: LD_PRELOAD and the dlsym recursion problem.
                                            
                                            
