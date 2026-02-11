# Why Your Asyncio Tasks Are Dying Silently in FastAPI

Async and await are powerful features in Python, but they come with their own set of pitfalls, especially when dealing with garbage collection. I recently encountered a frustrating bug in my FastAPI application where a background task would crash silently or complain about being destroyed while still pending. Here's what happened and how I fixed it.

## The Problem: "Task was destroyed but it is pending!"

I had a simple background task designed to listen to a Redis Pub/Sub channel and forward messages to a WebSocket (for a OneBot integration). I started this task in my FastAPI application's `lifespan` event.

### The Original Code

```python
# backend/main.py

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Start the background listener
    asyncio.create_task(onebot.onebot_redis_listener())

    yield

    # Clean up resources
    # ...
```

At first glance, this looks fine. `create_task` schedules the coroutine to run on the event loop. However, shortly after starting the server, I would see this error in the logs:

```
RuntimeError: coroutine ignored GeneratorExit
Task was destroyed but it is pending!
```

Or sometimes, the listener would just stop working without any error message at all.

## The Root Cause: Garbage Collection

The issue lies in how Python's garbage collector (GC) interacts with `asyncio.Task` objects.

When you call `asyncio.create_task()`, it returns a `Task` object. In my code above, I didn't assign this return value to any variable. Because no variable was holding a reference to the task, Python's reference count for the object dropped to zero.

Python's garbage collector (GC) is aggressive. It saw the `Task` object had no references and decided to reclaim the memory, effectively killing the task immediately, even though it was still running in the event loop!

This is a well-known "gotcha" in the Python asyncio documentation:

> **Important**: Save a reference to the result of this function, to avoid a task disappearing mid-execution. The event loop only keeps weak references to tasks. A task that is not referenced elsewhere may get garbage collected at any time, even before it is done.

## The solution: Keep a Strong Reference

The fix is simple: Assign the task to a variable that lives as long as your application loop.

### The Fixed Code

```python
# backend/main.py

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 1. Assign the task to a variable to keep a strong reference
    listener_task = asyncio.create_task(onebot.onebot_redis_listener())

    yield

    # 2. Gracefully cancel the task on shutdown
    if listener_task:
        listener_task.cancel()
        try:
            await listener_task
        except asyncio.CancelledError:
            pass # Task cancellation is expected
```

By assigning `listener_task = ...`, I ensure the task object has at least one strong reference, preventing the GC from killing it prematurely.

## Handling Clean Shutdowns

Solving the GC issue introduced a new problem: when I stopped the server, the task was still running, and the event loop would close while the task was active, leading to "Task was destroyed" warnings again.

To fix this, we need to explicitly handle the `asyncio.CancelledError` inside the background task itself.

```python
# backend/routers/onebot.py

async def onebot_redis_listener():
    try:
        async for message in pubsub.listen():
            # ... process message ...
    except asyncio.CancelledError:
        # Catch the cancellation signal so we can clean up
        logger.info("Listener cancelled, cleaning up...")
        raise
    finally:
        # Always close resources!
        await r.close()
```

## Summary

1.  **Always assign `asyncio.create_task` to a variable.** The event loop only holds a weak reference, so your task will be garbage collected if you don't hold a strong reference yourself.
2.  **Manage the lifecycle.** Cancel and await your background tasks in the shutdown phase of your `lifespan` function.
3.  **Handle Cancellation.** Ensure your long-running tasks catch `asyncio.CancelledError` or use `finally` blocks to close connections (like Redis or DB pools) properly.
