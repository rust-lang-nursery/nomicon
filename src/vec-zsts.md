# Handling Zero-Sized Types

It's time. We're going to fight the specter that is zero-sized types. Safe Rust
*never* needs to care about this, but Vec is very intensive on raw pointers and
raw allocations, which are exactly the two things that care about
zero-sized types. We need to be careful of two things:

* The raw allocator API has undefined behavior if you pass in 0 for an
  allocation size.
* raw pointer offsets are no-ops for zero-sized types, which will break our
  C-style pointer iterator.

Thankfully we abstracted out pointer-iterators and allocating handling into
RawValIter and RawVec respectively. How mysteriously convenient.




## Allocating Zero-Sized Types

So if the allocator API doesn't support zero-sized allocations, what on earth
do we store as our allocation? `Unique::empty()` of course! Almost every operation
with a ZST is a no-op since ZSTs have exactly one value, and therefore no state needs
to be considered to store or load them. This actually extends to `ptr::read` and
`ptr::write`: they won't actually look at the pointer at all. As such we never need
to change the pointer.

Note however that our previous reliance on running out of memory before overflow is
no longer valid with zero-sized types. We must explicitly guard against capacity
overflow for zero-sized types.

Due to our current architecture, all this means is writing 3 guards, one in each
method of RawVec.

```rust,ignore
impl<T> RawVec<T> {
    fn new() -> Self {
        // This branch should be stripped at compile time.
        let cap = if mem::size_of::<T>() == 0 {
            std::usize::MAX
        } else {
            0
        };
        // Unique::empty() doubles as "unallocated" and "zero-sized allocation"
        Self {
            ptr: Unique::empty(),
            cap,
        }
    }

    fn grow(&mut self) {
        let elem_size = mem::size_of::<T>();
        let align = mem::align_of::<T>();
        unsafe {
            // Since we set the capacity to `usize::MAX` for ZST is
            // 0, getting to here necessarily means the Vec is overfull.
            assert!(elem_size == 0, "capacity overflow");

            let layout;

            let (new_cap, ptr) = if self.cap == 0 {
                layout = alloc::Layout::from_size_align_unchecked(elem_size, align);
                let ptr = alloc::alloc(layout);
                (1, ptr)
            } else {
                let new_cap = self.cap * 2;
                layout = alloc::Layout::from_size_align_unchecked(self.cap * elem_size, align);
                let ptr = alloc::realloc(
                    self.ptr.as_ptr() as *mut u8,
                    layout,
                    new_cap * elem_size,
                );
                (new_cap, ptr)
            };

            // If allocate or reallocate fail, we'll get `null` back
            if ptr.is_null() {
                alloc::handle_alloc_error(layout);
            }

            self.ptr = Unique::new_unchecked(ptr as *mut T);
            self.cap = new_cap;
        }
    }
}

impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        let align = mem::align_of::<T>();
        let elem_size = mem::size_of::<T>();
        // don't free zero-sized allocations, as they were never allocated.
        if self.cap != 0 && elem_size != 0 {
            unsafe {
                alloc::dealloc(
                    self.ptr.as_ptr() as *mut u8,
                    alloc::Layout::from_size_align_unchecked(self.cap * elem_size, align),
                );
            }
        }
    }
}
```

That's it. We support pushing and popping zero-sized types now. Our iterators
(that aren't provided by slice Deref) are still busted, though.




## Iterating Zero-Sized Types

Zero-sized offsets are no-ops. This means that our current design will always
initialize `start` and `end` as the same value, and our iterators will yield
nothing. The current solution to this is to cast the pointers to integers,
increment, and then cast them back:

```rust,ignore
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        Self {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()) as *const T
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            },
        }
    }
}
```

Now we have a different bug. Instead of our iterators not running at all, our
iterators now run *forever*. We need to do the same trick in our iterator impls.
Also, our size_hint computation code will divide by 0 for ZSTs. Since we'll
basically be treating the two pointers as if they point to bytes, we'll just
map size 0 to divide by 1.

```rust,ignore
impl<T> Iterator for RawValIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = if mem::size_of::<T>() == 0 {
                    (self.start as usize + 1) as *const T
                } else {
                    self.start.offset(1)
                };
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let elem_size = if mem::size_of::<T>() == 0 {
            1
        } else {
            mem::size_of::<T>()
        };
        let len = (self.end as usize - self.start as usize) / elem_size;
        (len, Some(len))
    }
}

impl<T> DoubleEndedIterator for RawValIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = if mem::size_of::<T>() == 0 {
                    (self.end as usize - 1) as *const T
                } else {
                    self.end.offset(-1)
                };
                Some(ptr::read(self.end))
            }
        }
    }
}
```

And that's it. Iteration works!
