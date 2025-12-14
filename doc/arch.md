Arch 2025
=========
Eonil, 2025.

Basically, almost all programs are built in this way.

```
- model         Contains common data types.
  - ...
- op            Runs each feature operations.
  - new_chat    Make a new AI chat session.
  - submit_talk Submits a new talk to AI chat session.
  - clear_all   Deletes all AI chat session.
  - ...
- io            High level, per-service communication with external world.
  - repo        Ephemeral data lives in memory only. Fast but all data will be lost at the end of the session.
  - db          Persisted data over multiple sessions.     
    - config    
    - secret    Secret storage including auth.
    - market    Market data.
    - chat      Chatting data with AI.
    - cache
  - net
    - chatgpt   ChatGPT API
    - claude    Claude API
    - grok      Grok API
    - ...
  - ui          Communication gateway to shell.
  - ...
- sys           Low level infrastructure. Probably we don't need this layer if platform provides this in a good way.
  - net
    - fs
    - http
    - ws
    - ...
- util              Utility functions.
  - chan            Bridge channel.
  - ...
```

Each module has these key members.
- `sys::System`     A `trait` contains all low-level I/O subsystems.
- `sys::real`       `fn () -> impl sys::System`. Returns an opaque concrete type for actual low level I/O implementation.
- `sys::mock`       `fn () -> impl sys::System`. Returns an opaque concrete type for low level I/O test mocking.
- `io::IO`          A `trait` contains all high-level I/O to certain features.
- `io::real`        `fn () -> impl io::IO`. Returns an opaque concrete type for actual high level I/O implementation.
- `io::mock`        `fn () -> impl io::IO`. Returns an opaque concrete type for high level I/O test mocking.
- `ops::run`        `async fn (io: &impl IO, drain: Drain) -> ()`. This creates a steppable state machine.
- `util::chan`      `fn () -> (Queue, Drain)`. Creates `Queue` and `Drain` function pairs.



Dataflow
--------
- In main entry point...
  - App makes a queue/drain channel pair.
  - App makes system object, and makes I/O object with the system object and queue function.
  - App makes ops run object. (future) by passing drain function.
  - App loops ops run stepping on external signaling calls. (future polling) 
  - So basically, this is full "simulation" based architecture, completely a state machine.
- In ops run...
  - It dequeues app commands by calling drain function.
  - Processes the command by spawning a new coroutine context. (in-thread concurrency)
  - It spawns a new coroutine, and runs an async function for the op.
  - The op will be called with immutable reference to I/O object.
  - As I/O object is considered as gateway to remote devices, it's find to be full immutable.
  - I/O object is basically communication channel to external world.
  - Once it spaned a new flow, it yields execution. (future polling return)
  - Ops run is complete state machine. Therefore someone else has to step it manually.
  - Basically, incoming I/O signal triggers the external stepping.
  - All ops run from single entry point function. Consider this as the main function of the app.
  - Which means all spawned coroutines and subroutine calls will be stepped by stepping the root future object.
- All I/O is discrete as...
  - They are all async functions. All non-blocking, waiting and progresses only on poll.
  - Rust async runtime is responsible to poll. And basically, app only polls on explicit stepping.
  - On any incoming I/O readiness signal, app polls the main ops run. And it will step all app logics.



Machine-Facing vs. Human-Facing Apps
------------------------------------
- Above example is basic. That runs simulation stepping only on external I/O completion signals.
- That would be enough for machine-facing apps like servers.
- But human-facing apps needs a bit more complicated structure to support smooth human interactions.
- Essentially, human interacting apps needs smooth animation loop. Like 60Hz or 120Hz.
- And they must run on main thread on most platforms.
- Those are essential core difference. And the difference needs a new dataflow structure.
  - Human-facing part should step at regular period (e.g. 120Hz) and runs at firm real-time.
    - Frame-time spike is critical failure.
  - Machine-facing part should step only at I/O completion signals to be efficient.
    - Periodic polling is just a waste, and there's no concept of frame-time constraint.
- That means...
  - The app has to split into 2 independent dataflows.
  - It's not actually just 1 app. It's actually 2 apps in single package -- (1) shell app (2) kernel app
  - As they are 2 independent dataflows, it's best to make them isolated and communicate only by message passing.
  - App's main entry point function split dataflow into 2 threads -- (1) main (2) sub.
  - App runs shell app in main thread and kernel app in sub thread. As UI code must run in main thread in most systems.
  - Now the problem is, the communication layer. How should we divide app at which abstraction layer?
 


High-Level vs Low-Level Isolation
---------------------------------
- We split app into isolated dataflows running in isolated threads.
- We use thread as thread is the minimal unit of dataflow in modern systems.
- If isolation unit is thread, we can share memory easily for unlimited bandwidth. Although it still needs proper care.
- If isolation unit is process or VM, it would be limited, but we still can have some sharing and high bandwidth.
- But if it crosses machine border, there's no more cheap and near-unlimited bandwidth.
- Although special hardwares can provide better bandwidth, it's still slower than single machine.
- Also, if kernel and shell are made in heterogeneous systems, it won't be easy to share even for in-memory data.
  - e.g. Rust kernel + Swift/Kotlin/TypeScript shell.
- Bandwidth decreases as we seek for more flexibility.
  - With more flexibility, communication will be slow.
  - With less flexibility, communication will be fast.

By the way, in modern environment, most systems for human-facing apps are heterogeneous.
- As world is divided into a few major platform vendors and they all want to keep their own ecosystem.
- This is highly political issue, therefore cannot be solved forever until one company monopoly the market.
- And market monpoly is also not a good situation. As monopoly lowers overall quality and engineers will be suffered.
- Probably current state would be one of the ideal state. Competition between a few giants.
- Therefore, we have to assume that we have to support heterogeneous systems.
  - If everything is in heterogeneous system, we always need a middle layer abstraction.
- In heterogeneous system, there's no cheap way for unlimited bandwidth. Except sharing fixed layout memory.
  - Heterogeneous system means, their memory layout are all different.
  - Reading/writing data from other system always require some kind of "translation".
  - And translation is always super expensive as it has to involve O(n) copy operation.
- Also, abstraction layer should be at low level. Because...
  - Inter-system communication is expensive.
  - Protocol has to be stable and reliable.
  - Moving target is super expensive to keep. If protocol constantly changes? No one can pay for it.
  - High-level models are always changing. If we make protocol at high-level, we have to pay huge just to keep it work.
  - Low-level models do not change frequently. If we make protocol at low-level, it will be stable and reliable at lower cost.









