Launch Tasks from Tasks
=======================

Sometimes it is convenient to launch tasks from other tasks.
For example you may not know what computations to run until you have the
results of some initial computations.

Motivating example
------------------

We want to download one piece of data and turn it into a list.  Then we want to
submit one task for every element of that list.  We don't know how long the
list will be until we have the data.

So we send off our original ``download_and_convert_to_list`` function, which
downloads the data and converts it to a list on one of our worker machines:

.. code-block:: python

   future = e.submit(download_and_convert_to_list, uri)

But now we need to submit new tasks for individual parts of this data.  We have
three options.

1.  Gather the data back to the local process and then submit new jobs from the
    local process
2.  Gather only enough information about the data back to the local process and
    submit jobs from the local process
3.  Submit a task to the cluster that will submit other tasks directly from
    that worker

Gather the data locally
-----------------------

If the data is not large then we can bring it back to the client to perform the
necessary logic on our local machine:

.. code-block:: python

   >>> data = future.result()                  # gather data to local process
   >>> data                                    # data is a list
   [...]

   >>> futures = e.map(process_element, data)  # submit new tasks on data
   >>> analysis = e.submit(aggregate, futures) # submit final aggregation task

This is straightforward and, if ``data`` is small then it is probably the
simplest, and therefore correct choice.  However, if ``data`` is large then we
have to choose another option.


Submit tasks from client
------------------------

We can run small functions on our remote data to determine enough to submit the
right kinds of tasks.  In the following example we compute the ``len`` function
on ``data`` remotely and then break up data into its various elements.

.. code-block:: python

   >>> n = e.submit(len, data)                 # compute number of elements
   >>> n = n.result()                          # gather n (small) locally

   >>> from operator import getitem
   >>> elements = [e.submit(getitem, data, i) for i in range(n)]  # split data

   >>> futures = e.map(process_element, elements)
   >>> analysis = e.submit(aggregate, futures)

We compute the length remotely, gather back this very small result, and then
use it to submit more tasks to break up the data and process on the cluster.
This is more complex because we had to go back and forth a couple of times
between the cluster and the local process, but the data moved was very small,
and so this only added a few milliseconds to our total processing time.


Submit tasks from worker
------------------------

*Note: this interface is new and experimental.  It may be changed without
warning in future versions.*

Alternatively we submit tasks from other tasks.  This allows us to make
decisions while on worker nodes.  To do this we will make a new client object
on the worker itself.  There is a convenience function for this that will
connect you to the correct scheduler that the worker is connected to.

.. code-block:: python

   from distributed import local_executor

   def process_all(data):
       with local_executor() as e:
           elements = e.scatter(data)
           futures = e.map(process_element, elements)
           analysis = e.submit(aggregate, futures)
           result = analysis.result()
       return result

    analysis = e.submit(process_all, data)  # spawns many tasks

This approach is more complex, but very powerful.  It allows you to spawn tasks
that themselves act as potentially long-running clients, managing their own
independent workloads.

Extended Example
~~~~~~~~~~~~~~~~

This example computing the fibonacci numbers creates tasks that submit tasks
that submit tasks that submit other tasks, etc..

```python
In [1]: from distributed import Executor, local_executor

In [2]: e = Executor()

In [3]: def fib(n):
   ...:     if n < 2:
   ...:         return n
   ...:     else:
   ...:         with local_executor() as ee:
   ...:             a = ee.submit(fib, n - 1)
   ...:             b = ee.submit(fib, n - 2)
   ...:             a, b = ee.gather([a, b])
   ...:             return a + b
   ...:

In [4]: future = e.submit(fib, 100)

In [5]: future
Out[5]: <Future: status: finished, type: int, key: fib-7890e9f06d5f4e0a8fc7ec5c77590ace>

In [6]: future.result()
Out[6]: 354224848179261915075
```

Technical details
~~~~~~~~~~~~~~~~~

Tasks that invoke ``local_executor`` are conservatively assumed to be
*long running*.  They can take a long time blocking, waiting for other tasks to
finish.  In order to avoid having them take up processing slots the following
actions occur whenever a task invokes ``local_executor``.

1.  The thread on the worker that runs this functions *secedes* from the thread
    pool and goes off on its own.  This allows the thread pool to populate that
    slot with a new thread and continue processing tasks without counting this
    long running task against its normal quota.
2.  The Worker sends a message back to the scheduler temporarily increasing its
    allowed number of tasks by one.  This likewise lets the scheduler allocate
    more tasks to this worker, not counting this long running task against it.

Because of this behavior you can happily launch long running control tasks that
manage worker-side clients happily, without fear of deadlocking the cluster.
