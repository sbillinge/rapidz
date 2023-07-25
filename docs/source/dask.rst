Dask Integration
================

The ``rapidz.dask`` module contains a Dask_-powered implementation of the
core Stream object.  This is a drop-in implementation, but uses Dask for
execution and so can scale to a multicore machine or a distributed cluster.


Quickstart
----------

.. currentmodule:: rapidz

Installation
++++++++++++

First install dask and dask.distributed::

   conda install dask
   or
   pip install dask[complete] --upgrade

You may also want to install Bokeh for web diagnostics::

   conda install -c bokeh bokeh
   or
   pip install bokeh --upgrade

Start Local Dask Client
+++++++++++++++++++++++

Then start a local Dask cluster

.. code-block:: python

   from dask.distributed import Client
   client = Client()

This operates on a local processes or threads.  If you have Bokeh installed
then this will also start a diagnostics web server at
http://localhost:8787/status which you may want to open to get a real-time view
of execution.

Sequential Execution
++++++++++++++++++++

.. autosummary::
   Stream.emit
   map
   sink

Before we build a parallel stream, lets build a sequential stream that maps a
simple function across data, and then prints those results.  We use the core
``Stream`` object.

.. code-block:: python

   from time import sleep

   def inc(x):
       sleep(1)  # simulate actual work
       return x + 1

   from rapidz import Stream

   source = Stream()
   source.map(inc).sink(print)

   for i in range(10):
       source.emit(i)

This should take ten seconds we call the ``inc`` function ten times
sequentially.

Parallel Execution
++++++++++++++++++

.. currentmodule:: rapidz

.. autosummary::
   scatter
   buffer

.. currentmodule:: rapidz.dask

.. autosummary::
   gather

That example ran sequentially under normal execution, now we use ``.scatter()``
to convert our stream into a DaskStream and ``.gather()`` to convert back.

.. code-block:: python

   source = Stream()
   source.scatter().map(inc).buffer(8).gather().sink(print)

   for i in range(10):
       source.emit(i)

You may want to look at http://localhost:8787/status during execution to get a
sense of the parallel execution.

This should have run much more quickly depending on how many cores you have on
your machine.  We added a few extra nodes to our stream, lets look at what they
did.

-   ``scatter``: Converted our Stream into a DaskStream.  The elements that we
    emitted into our source were sent to the Dask client, and the subsequent
    ``map`` call used that client's cores to perform the computations.
-   ``gather``: Converted our DaskStream back into a Stream, pulling data on our
    Dask client back to our local stream
-   ``buffer(5)``: Normally gather would exert back pressure so that the source
    would not accept new data until results finished and were pulled back to the
    local stream.  This back-pressure would limit parallelism.  To counter-act
    this we add a buffer of size eight to allow eight unfinished futures to
    build up in the pipeline before we start to apply back-pressure to
    ``source.emit``.

.. _Dask: https://dask.pydata.org/en/latest/


HPC Execution
+++++++++++++
If ``dask-jobqueue`` is installed you can use high performance computing (HPC)
resources to execute the tasks.

.. code-block:: python

   from dask.distributed import Client
   from dask_jobqueue import PBSCluster
   cluster = PBSCluster()
   cluster.scale(10)  # Ask for ten workers

   client = Client(cluster)

   # Now this pipeline runs using HPC resources
   source = Stream()
   source.scatter().map(inc).buffer(8).gather().sink(print)

   for i in range(10):
       source.emit(i)

For more details on using dask-jobqueue_ please see the associated docs.

.. _dask-jobqueue: https://dask-jobqueue.readthedocs.io/en/latest/