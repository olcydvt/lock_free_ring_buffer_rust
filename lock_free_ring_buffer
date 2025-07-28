pub mod lock_free_ring_buffer {

    use std::cell::UnsafeCell;
    use std::sync::atomic::{AtomicU32, Ordering};

    const fn is_power_of_two(n: usize) -> bool {
        n > 0 && (n & (n - 1)) == 0
    }

    pub struct RingBuffer<T, const SIZE: usize> {
        buffer: [UnsafeCell<T>; SIZE], // Buffer storage
        write_cursor: AtomicU32,       // Write cursor
        read_cursor: AtomicU32,        // Read cursor
    }

    // SAFETY: Thread-safe when T is Send because:
    // - Atomic cursors handle concurrent access
    // - Power-of-two size prevents slot contention
    // - UnsafeCell provides safe interior mutability
    unsafe impl<T: Send, const SIZE: usize> Sync for RingBuffer<T, SIZE> {}

    // Implement Default for RingBuffer
    impl<T, const SIZE: usize> RingBuffer<T, SIZE>
    where
        T: Default,
        [T; SIZE]: Sized,
    {
        const BUFFER_MASK: u32 = (SIZE - 1) as u32;

        pub fn new() -> Self {
            const {
                assert!(is_power_of_two(SIZE), "Size must be a power of two");
            }
            RingBuffer {
                write_cursor: AtomicU32::new(0),
                read_cursor: AtomicU32::new(0),
                buffer: std::array::from_fn(|_| UnsafeCell::new(T::default())),
            }
        }

        pub fn try_write(&self, item: T) -> bool {
            let curr_write_curs: u32 = self.write_cursor.load(Ordering::Relaxed);

            loop {
                let curr_read_curs: u32 = self.read_cursor.load(Ordering::Acquire);
                let next_write_curs: u32 = (curr_write_curs + 1) & Self::BUFFER_MASK;

                // Check if the buffer is full
                if next_write_curs == curr_read_curs {
                    return false; // Buffer is full
                }

                // Attempt to write the item
                if self
                    .write_cursor
                    .compare_exchange_weak(curr_write_curs, next_write_curs, Ordering::AcqRel, Ordering::Relaxed)
                    .is_ok()
                {
                    unsafe {
                        *self.buffer[curr_write_curs as usize].get() = item; // Write the item
                    }
                    return true; // Write successful
                }
            }
        }

        pub fn try_read(&self) -> Option<T> {
            let curr_read_curs: u32 = self.read_cursor.load(Ordering::Relaxed);

            loop {
                let curr_write_curs: u32 = self.write_cursor.load(Ordering::Acquire);
                if curr_read_curs == curr_write_curs {
                    return None; // Buffer is empty
                }

                // Attempt to read the item
                if self
                    .read_cursor
                    .compare_exchange_weak(
                        curr_read_curs,
                        (curr_read_curs + 1) & Self::BUFFER_MASK,
                        Ordering::AcqRel,
                        Ordering::Relaxed,
                    )
                    .is_ok()
                {
                    return Some(unsafe { self.buffer[curr_read_curs as usize].get().read() });
                }
            }
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn single_threaded_write_read() {
            let buffer: RingBuffer<i32, 4> = RingBuffer::new();
            assert!(buffer.try_write(1));
            assert!(buffer.try_write(2));
            assert_eq!(buffer.try_read(), Some(1));
            assert_eq!(buffer.try_read(), Some(2));
            assert_eq!(buffer.try_read(), None);
        }

        #[test]
        fn buffer_full() {
            let buffer: RingBuffer<i32, 4> = RingBuffer::new();
            assert!(buffer.try_write(1));
            assert!(buffer.try_write(2));
            assert!(buffer.try_write(3)); 
            assert!(!buffer.try_write(4));// Buffer should be full
            assert_eq!(buffer.try_read(), Some(1));
            assert!(buffer.try_write(3)); // Should succeed after reading
            assert_eq!(buffer.try_read(), Some(2));
            assert_eq!(buffer.try_read(), Some(3));
        }
    }
}
