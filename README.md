# Test Main

## Usage

```rust
mod ring_buffer;

use ring_buffer::lock_free_ring_buffer::RingBuffer;
use std::sync::Arc;
use std::thread;

fn main() {
    const BUFFER_SIZE: usize = 4;
    const NUM_ITEMS: usize = 10;

    let rb = Arc::new(RingBuffer::<usize, BUFFER_SIZE>::new());

    let producer = {
        let rb = rb.clone();
        thread::spawn(move || {
            for i in 0..NUM_ITEMS {
                loop {
                    if rb.try_write(i) {
                        break; // Successfully wrote item
                    }
                }
                println!("[Producer] Wrote: {}", i);
            }
        })
    };

    let consumer = {
        let rb = rb.clone();
        thread::spawn(move || {
            let mut received = 0;
            while received < NUM_ITEMS {
                if let Some(val) = rb.try_read() {
                    assert!(val < NUM_ITEMS); // Verify valid data
                    println!("[Consumer] Read: {}", val);
                    received += 1;
                } else {
                    continue;
                }
            }
        })
    };

    producer.join().unwrap();
    consumer.join().unwrap();

    println!("Successfully passed {} items through buffer!", NUM_ITEMS);
}


```

## Contributing

Pull requests are welcome. For major changes, please open an issue first
to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License

[MIT](https://choosealicense.com/licenses/mit/)

```

## Contributing

Pull requests are welcome. For major changes, please open an issue first
to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License

[MIT](https://choosealicense.com/licenses/mit/)
