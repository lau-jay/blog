+++
title = "Pickle Error"
date = 2017-10-17T13:52:34+08:00
images = []
tags = ["python"]
categories = ["python"]
draft = false
+++
### Error Message

```
cPickle.PicklingError: Can't pickle

```

### shell

```bash
python3 -m trace -tg --ignore-dir /Library $1
```

### bug code

```python
def process_way():
    workers = 10
    with futures.ProcessPoolExecutor(workers) as executor:
        futs = {executor.submit(blocking_way) for i in range(10)}
    return len([fut.result() for fut in futs])
```

### Reason
```
Pool methods all use a queue.Queue to pass tasks to the worker processes.
Everything that goes through the queue.Queue must be pickable.
So, multiprocessing can only transfer Python objects to worker processes which can be pickled.
Functions are only picklable if they are defined at the top-level of a module, bound methods are not picklable.
```

### Fixed

```python
def process_way():
    workers = 10
    with futures.ThreadPoolExecutor(workers) as executor:
        futs = {executor.submit(blocking_way) for i in range(10)}
    return len([fut.result() for fut in futs])
```
