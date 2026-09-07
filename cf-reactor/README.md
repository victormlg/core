
# Moving stuff around

Rather than exposing the raw file-descriptor bookkeeping required for `select(2)`, we introduce a unified interface that serves both the reactor-plugin and event-driven code paths. This is achieved by encapsulating all relevant state in a context struct, `ReactorContext`:

```C
typedef struct ReactorContext
{
    int *all_fds; // heap allocated array of fds
    size_t all_fds_capacity; // total number of fds. number of nova fds + number of event fds 
    size_t num_nova_fds; // this is returned by the reactor-plugin
    size_t num_fds; // this is 1
    // the first (num_nova_fds - 1) slots in the array are reserved for the reactor-plugin, the last one is reserved for the event driven code.

    fd_set readfds;
} ReactorContext;
```

- `ReactorContextInitialize()`: initializes the reactor-plugin and event-driven code. Wraps `ReactorNovaInitialize()`
- `ReactorContextSetupFileDescriptors()`: populates readfds with the file descriptors to monitor, prior to the select() call.
- `ReactorContextHandleEvents()`: iterates over the file descriptors and dispatches the appropriate action based on which ones were signaled as ready. Wraps `ReactorNovaHandleTimeout` and `ReactorNovaHandleEvents()`.
- `ReactorContextFinalize()`: releases the daemon's associated resources. Wraps `ReactorNovaFinalize()`.

# Tracking spec & Events

In order to track all the events promises, we use two datastructures: a global list of `"Watcher"`, which is a struct associated with an event type and the promise name (also called `key`) and a global hashmap mapping this `key` to a `bundle` which is parsed from the policy.

On an agent run, cf-agent makes cf-reactor read the policy, and rebuilds the list of watchers and the hashmap using the single function `WatcherRegister(key, event_type, payload, bundle, interval)`. Each events promise is associated with an event type, which is defined in `when` bodies:

```cf3
body when file_deleted(filename)
{
    file_deleted => "$(filename)";
}
```

Every event type is must have defined:
- A check function (called `check_fn` in `Watcher`): This is a function defined specifically for the event that checks if the conditions holds. For example, in case of file deletion, we check if the file doesn't exist anymore compare to the last time we checked. If yes, then it returns `true`.
- A state (called `"payload"`): This is a struct whose interpretation depends on the event type (thus being declared as `void *`). We typically need some state that we compare between each event-check. In the case of file deletion, we need to know the name of the file we are watching, and whether the file existed last time we checked.
- A payload destroying function (called `destroy_payload`): This is simply a function to free the state associated with the event type.

Also, we need a function that will create the state. That's what `FileWatcherPayloadNew()` does. 

So each event type we add in the future just need to have these four things defined, and we need to create the `Watcher` object with the right functions inside `WatcherRegister()` and also call the right `"...PayloadNew()"` function.


# Polling & Running bundles

`ReactorContextInitialize()` sets up all the necessary data structures for polling, and then starts `WatcherThreadMain`, which polls for events as follows:

- It iterates through each watcher in the global list of watchers.
- If the elapsed time exceeds the watcher's `interval`, it runs `check_fn` to determine whether an event has been triggered.
- If an event was triggered, it pushes the watcher's `key` (the promise name) onto a thread-safe queue, then signals the file descriptor via `WakeupChannelNotify`, which `select(2)` will pick up on its next iteration. It then goes back to sleep.

In parallel, `EventWatcherHandleEvents`, called from within `ReactorContextHandleEvents`, reads from the file descriptor with `WakeupChannelReadFd()` once notified that an event has occurred, and pops the thread-safe queue until it's empty. Each key popped from the queue is looked up in the global hashmap to retrieve the corresponding bundle, which `cf-reactor` then runs (in another thread or subprocess)
