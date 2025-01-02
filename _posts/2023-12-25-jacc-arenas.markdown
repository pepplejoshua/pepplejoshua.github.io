---
layout: post
title:  "trying to understand arena allocators"
date:   2023-12-25 23:42:22 -0700
categories: programming compilers c
---

# with chatgpt and some festive reasoning

While reading chapter 2 of "A Retargetable C Compiler Design and Implementation" by Christopher W. Fraser, David R. Hanson,
I came across some allocator code which I have written in [alloc.c](https://github.com/pepplejoshua/jacc/blob/master/src/alloc.c)
a few things have confused me. But with a combination of
[this wonderful blog post](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator) by Ryan Fleury,
[ChatGPT-4 conversation](https://chat.openai.com/share/cbc113ed-e06f-40bc-be80-dbd3765dcff5) and actually thinking
about what the code meant, I have come to a better understanding of this code:

```c
void *allocate(unsigned long n, unsigned a) {
  struct mem_block *ap = arenas[a];
  // round up n to align it properly in memory
  n = roundup(n, sizeof(union align));
  while (ap->avail + n > ap->limit) {
    // get new block from either free blocks or by allocating one
    ap->next = free_blocks;
    if (ap->next != NULL) { // there is at least 1 free mem block
      free_blocks = free_blocks->next; // move the first free block to the next free block
      ap = ap->next; // move the ap pointer to the new block
    } else {
      // allocate new mem block
      // sizeof(union header) is the size of the header of the block
      // the header stores metadata about the block like the size of the block,
      // the next block, etc.
      unsigned m = sizeof(union header) + n + 10 * 1024; // 24 bytes (on my 64-bit computer) + n bytes + 10240 bytes
      ap->next = malloc(m); // allocate the memory
      ap = ap->next; // move the ap pointer to the new block
      if (ap == NULL) {
        error("insufficient memmory\n");
        exit(1);
      }
      ap->limit = (char *)ap + m; // set the limit of the block to starting addr + size of block
    }
    // set the avail of the block to the point right after the header of the block
    // the header is of size sizeof(union header)
    ap->avail = (char *)((union header *)ap + 1);
    ap->next = NULL; // set the next of the block to NULL since it is the last block
    arenas[a] = ap; // set the arena to the new block
  }
  ap->avail += n; // move the avail pointer to the next free location
  return ap->avail - n; // return the location of the allocated memory
}

void deallocate(unsigned a) {
  while (arenas[a]->next != NULL) { // while there are more blocks in the arena
    arenas[a] = arenas[a]->next; // move the arena pointer to the next block
  }
  // set the next pointer of the last block to the first free block
  arenas[a]->next = free_blocks;
  // let the free blocks now start from the first block of this arena, effectively
  // freeing all the blocks in the arena for reuse
  free_blocks = first[a].next;
  // set the next of the first block to NULL since it is the last block
  first[a].next = NULL;
  // set the arena to the first block
  arenas[a] = &first[a];
}
```

Let me preface with:

- I have never used a memory allocator that is not `new` and `free`, and
- I have never used C/C++ to do a lot of the "magic" in this code before.

So there is a lot going on with this code above. Initially, I thought `struct mem_block` would hold the memory directly (I am not sure how to explain this. Like it would `malloc` an array of the memory or a `char *` and do stuff with it?). Instead, it imprints onto the memory, making itself a part of the memory it is managing. Very neat (and confusing at first). The memory is allocated using `malloc` (where `m` is the size of the entire memory block of the struct mem_block, part of which is allocatable [mem_block.limit - mem_block.avail]). The struct pointer is set to the start of the memory block and the size of the memory block, `m`, is derived from adding the size of the header (we are accounting for space for the different pointers we will imprint to manage the memory block) + `n`, which is the memory-aligned number of bytes we need to allocate + 10,240 extra bytes for future allocations. This idea of a `header` is still confusing.

After allocating _m_ bytes with `malloc` and setting the current arena to the start of the memory block, we can easily get the limit of this memory block by adding `m` to the start of the memory block (after casting it to a `char *`, which is essentially making it a pointer to a char, which is a byte). Adding `m` to a `char *` will move it `m` bytes forward, which is the limit of the memory block. Again, very neat. Now we need to compute the second member of `struct mem_block`, `avail`. This is done by adding the size of the header to the start of the memory block. This is because the header is the first thing in the memory block, so adding the size of the header to the start of the memory block will give us the start of the allocatable memory. We then set the `avail`pointer to this location. We then set the`next`pointer to`NULL` since this is the last block in the arena. We then set the arena to this block.

After writing all the above and then reasoning about deallocate(), I then came to the realization that this beautiful abstraction is actually 3D.

```c
static struct mem_block first[] = { { NULL }, { NULL }, { NULL } }, *arenas[] = { &first[0], &first[1], &first[2] };
```

The above piece of code lets us imagine that `first` will always be 3 individual doors and `arenas` will give you pointer indirection to those doors (at first). When you open an arena to allocate for the first time (essentially doing `arenas[a]` in allocate()), you will find nothing (since it is just a misdirection to `first`) and will need to make a new block. This new block will be allocated and stored in next (which means `first[a].next` also now points to something but its remaining fields are still `NULL` and will remain so).

I think this will all read a bit vague until you do some rough hand simulation of what the code does. Here is my best effort at doing so:

```python
# before any allocations
first = [NULL, NULL, NULL] # 3 empty blocks
arenas = [NULL, NULL, NULL] # 3 empty blocks from first
free_blocks = NULL # no free blocks

# after allocating 3 blocks to arenas[0], before any deallocates
first = [*0, NULL, NULL] # first[0].next points to the first block ever allocated to arenas[0] which is *0
# arenas[0] is the 3rd block belonging to the first arena or *2 since we are allocating space from it
arenas = [*2, NULL, NULL]
*0 = [*1, 10272, 10272] # 10272 is the size of the block, and this is the first block pointed to by first[0]
*1 = [*2, 10272, 10272] # 10272 is the size of the block
*2 = [NULL, 8000, 10272] # 10272 is the size of the block
free_blocks = NULL

# after deallocating arenas[0]
first = [NULL, NULL, NULL] # 3 empty blocks, since all 3 blocks from arenas[0] are now free
arenas = [NULL, NULL, NULL] # 3 empty blocks from first
free_blocks = *0 # the first block from arenas[0] is now free
*0 = [*1, 10272, 10272] # 10272 is the size of the block
*1 = [*2, 10272, 10272] # 10272 is the size of the block
*2 = [NULL, 8000, 10272] # 10272 is the size of the block

# after allocating 2 blocks to arenas[1], before any deallocates
first = [NULL, *3, NULL] # first[1].next points to the first block ever allocated to arenas[1] which is *3
# arenas[1] is the 2nd block belonging to the first arena or *4 since we are allocating space from it
arenas = [NULL, *3, NULL]
*3 = [*4, 10272, 10272] # 10272 is the size of the block
*4 = [NULL, 8000, 10272] # 10272 is the size of the block. NOTE: it has no next pointer since it is the last block
free_blocks = *0 # the first block from arenas[0] is now free and can be reused
*0 = [*1, 10272, 10272] # 10272 is the size of the block
*1 = [*2, 10272, 10272] # 10272 is the size of the block
*2 = [NULL, 8000, 10272] # 10272 is the size of the block

# after deallocating arenas[1]
first = [NULL, NULL, NULL]
arenas = [NULL, NULL, NULL]
free_blocks = *3 # the first block from arenas[1] is now free
*3 = [*4, 10272, 10272] # 10272 is the size of the block
*4 = [*0, 8000, 10272] # 10272 is the size of the block. NOTE: it's next pointer is now the former first block of free_blocks
*0 = [*1, 10272, 10272] # 10272 is the size of the block
*1 = [*2, 10272, 10272] # 10272 is the size of the block
*2 = [NULL, 8000, 10272] # 10272 is the size of the block
```

Ignore the fact that the code could have potentially reused block `*2` from `arenas[0]` instead of allocating new ones. I just wanted to show how the pointers are being manipulated. I think this is the best way to understand the code. `first[i]` will always be our grounding point (to reset to and beginning allocations from) while `arenas[i]` will always move across different blocks within its confines. Deallocating will extend `free_block`'s list with all the blocks from `arenas[i]`, reset `arenas[i]` back to `first[i]` and reset both their `next` pointers to `NULL`. I feel like the case for taking the first block from `free_blocks` and reusing it can be easily reasoned about from here since you just grab the first block from `free_blocks` and set it as the next block for `arenas[i]` and then set `free_blocks` to the next block of the block you just grabbed from `free_blocks`. You also reset the avail pointer to immediately after the header of the block you just grabbed from `free_blocks` and set
the `next` pointer of the block you just grabbed from `free_blocks` to `NULL` since it is now the last block in `arenas[i]`.

I dare say this is the most beautiful piece of code I have ever read and understood.

*jp*
