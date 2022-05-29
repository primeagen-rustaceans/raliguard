# Rate limit guard
Lazy rate limit semaphore (a.k.a fixed window algorithm without queueing) implementation to control your asynchronous code frequency execution

***
__Documentation__: [lib.rs/raliguard](lib.rs/raliguard)

## Overview
```rust
use std::{thread, sync, time};

use raliguard::Semaphore;


// Create a semaphore with restriction `5 tasks per 1 second`
let originl_sem = Semaphore::new(5, time::Duration::from_secs(1));

// Make it sharable between treads (or you can share between tasks)
let shared_sem = sync::Arc::new(
    sync::Mutex::new(originl_sem)
);

// This is a counter that increments when a thread completed
let shared_done_count = sync::Arc::new(sync::Mutex::new(0));

// Spawn 10 threads
for _ in 0..10 {
    let cloned_sem = shared_sem.clone();
    let cloned_done_state = shared_done_count.clone();
    let thread = thread::spawn(move || {
        let mut local_sem = cloned_sem.lock().unwrap();

        // Get required delay
        let calculated_delay = local_sem.calc_delay();
        drop(local_sem);

        // If delay exists, sleep it
        if let Some(delay) = calculated_delay {
            dbg!(&delay);
            thread::sleep(delay);
        }

        // Mark the thread is done
        let mut local_done_count = cloned_done_state.lock().unwrap();
        *local_done_count += 1;

    });
}

// So sleep 1 second
thread::sleep(time::Duration::new(1, 0));
let cloned_done_count = shared_done_count.clone();
let current_done = cloned_done_count.lock().unwrap();

// And then maximum 5 threads should be completed
// after 1 second sleeping
// (<= for clocks infelicity)
assert_eq!(*current_done <= 5, true);
```

## Features
* Calculated delay for sleep can be used with any async/await runtime backend or with threads
* Minimum memory used to save calls data
