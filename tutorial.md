[TOC]

# Core

The structure of this tutorial assumes an intermediate level
knowledge of Python but not much else. No knowledge of
concurrency is expected. The goal is to give you
the tools you need to get going with gevent and use it to solve
or speed up your applications today.

The primary pattern provided by gevent is the <strong>Greenlet</strong>, a
lightweight coroutine provided to Python as a C extension module.
Greenlets all run inside of the OS process for the main
program but are scheduled cooperatively by libev. This differs from
<a href="http://docs.python.org/library/subprocess.html">subprocceses</a> 
which are new processes are spawned by the OS.

## Greenlets

## Synchronous & Asynchronous Execution

The core idea of concurrency is that a larger task can be broken
down into a collection of subtasks whose operation does not
depend on the other tasks and thus can be run 
*asynchronously* instead of one at a time 
*synchronously*. A switch between the two
executions is known as a *context swtich*.

A context switch in gevent done through
*yielding*. In this case example we have
two contexts which yield to each other through invoking 
``gevent.sleep(0)``.

<pre>
<code class="python">import gevent

def foo():
    print 'Running in foo'
    gevent.sleep(0)
    print 'Emplict context switch to foo again'

def bar():
    print 'Emplict context to bar'
    gevent.sleep(0)
    print 'Implicit swtich switch back to bar'

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
</code>
</pre>

A somewhat synthetic example defines a ``task`` function
which is *non-deterministic*
(i.e. its output is not guaranteed to give the same result for
the same inputs). In this case the side effect of running the
function is that the task pauses its execution for a random
number of seconds.

<pre>
<code class="python">import gevent
import random

def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0,2))
    print 'Task', pid, 'done'

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for in xrange(10]
    gevent.joinall(threads)

print 'Synchronous:'
synchronous()

print 'Asynchronous:'
asynchronous()
</code>
</pre>

In the synchronous case all the tasks are run sequentially,
which results in the main programming *blocking* (
i.e. pausing the execution of the main program )
while each task executes.

The important parts of the program are the
``gevent.spawn`` which wraps up the given function
inside of a Greenlet thread. The list of initialized greenlets 
are stored in the array ``threads`` which is passed to
the ``gevent.joinall`` function which blocks the current
program to run all the given greenlets. The execution will step
forward only when all the greenlets terminate.

<p>The output is:</p>

<pre>
<code>
Synchronous:
Task 1 done
Task 2 done
Task 3 done
Task 4 done
Task 5 done
Task 6 done
Task 7 done
Task 8 done
Task 9 done
Task 10 done
Asynchronous:
Task 2 done
Task 3 done
Task 5 done
Task 10 done
Task 8 done
Task 6 done
Task 9 done
Task 1 done
Task 4 done
Task 7 done
</code>
</pre>

The important fact to notice is that the order of execution in
the async case is essentially random and that the total execution
time in the async case is much less than the sync case. In fact
the maximum time for the synchronous case to complete is when
each tasks pauses for 2 seconds resulting in a 20 seconds for the
whole queue. In the async case the maximum runtime is roughly 2
seconds since none of the tasks block the execution of the
others.

A more common use case, fetching data from a server
asynchronously, the runtime of ``fetch()`` will differ between
requests given the load on the remote server.

<pre><code class="python">import gevent.monkey
gevent.monkey.patch_socket()

import gevent
import urllib2
import simplejson as json

def fetch(pid):
    response = urllib2.urlopen('http://json-time.appspot.com/time.json')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print 'Process ', pid, datetime
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print 'Synchronous:'
synchronous()

print 'Asynchronous:'
asynchronous()
</code>
</pre>


## Race Conditions

The perennial problem involved with concurrency is known as a
*race condition*. Simply put is when two concurrent threads
/ processes depend on some shared resource but also attempt to
modify this value. This results in resources whose values become
time-dependent on the execution order. This is a problem, and in
general one should very much try to avoid race conditions since
they result program behavior which is globally
non-deterministic.*

One approach to avoiding race conditions is to simply not
have any global *state* shared between threads. To
communicate threads instead pass stateless messages between each 
other.

## Spawning Threads

gevent provides a few wrappers around Greenlet initialization.
Some of the most common patterns are:

<pre>
<code class="python">import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Each thread will be passed the message, and n arguments
    in its initialization.
    """
    gevent.sleep(n)
    print message

# Initialize a new Greenlet instance running the named function
# foo
thread1 = Greenlet.spawn(foo, "Hello", 1)
thread1.start()

# Wrapper for creating and runing a new Greenlet from the named 
# function foo, with the passd arguments
thread2 = gevent.spawn(foo, "I live!", 2)

# Lambda expressions
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Block until all threads complete.
gevent.joinall(threads)
</code>
</pre>

In addition to using the base Greenlet class, you may also subclass
Greenlet class and overload the ``_run`` method.

<pre>
<code class="python">from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print self.message
        gevent.sleep(self.n)

g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
</code>
</pre>


## Greenlet State

Like any other segement of code Greenlets can fail in various
ways. A greenlet may fail throw an exception, fail to halt or
consume too many system resources.</p>

<p>The internal state of a greenlet is generally a time-dependent
parameter. There are a number of flags on greenlets which let
you monitor the state of the thread</p>

- ``started`` -- Boolean, indicates whether the Greenlet has been started. </li>
- ``ready()`` -- Boolean, indicates whether the Greenlet has halted</li>
- ``successful()`` -- Boolean, indicates whether the Greenlet has halted and not thrown an exception</li>
- ``value`` -- arbitrary, the value returned by the Greenlet</li>
- ``exception`` -- exception, uncaught exception instance thrown inside the greenlet</li>

<pre>
<code class="python">import gevent

def win():
    return 'You win!'

def fail():
    raise Exception('You fail at failing.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print winner.started # True
print loser.started  # True

# Exceptions raised in the Greenlet, stay inside the Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print 'This will never be reached'

print winner.value # 'You win!'
print loser.value  # None

print winner.ready() # True
print loser.ready()  # True

print winner.successful() # True
print loser.successful()  # False

# The exception raised in fail, will not propogate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.

print loser.exception

# It is possible though to raise the exception again outside
raise loser.exception
# or with
loser.get()
</code>
</pre>

## Program Shutdown

Greenlets that fail to yield when the main program receives a
SIGQUIT may hold the program's execution longer than expected.
This results in so called "zombie processes" which need to be
killed from outside of the Python interpreter.

A common pattern is to listen SIGQUIT events on the main program
and to invoke ``gevent.shutdown`` before exit.

<pre>
<code class="python">import gevent
import signal

def run_forever():
    gevent.sleep(1000)

if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.shutdown)
    thread = gevent.spawn(run_forever)
    thread.join()
</code>
</pre>

## Timeouts

Timeouts are a constraint on the runtime of a block of code or a
Greenlet.

<pre>
<code class="python">
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print 'Could not complete'

</code>
</pre>

Or with a context manager in a ``with`` a statement.

<pre>
<code class="python">import gevent
from gevent import Timeout

time_to_wait = 5 # seconds

class TooLong(Exception):
    pass

with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)
</code>
</pre>

In addition, gevent also provides timeout arguments for a
variety of Greenlet and data stucture related calls. For example:

<pre>
<code class="python">import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()

thread1 = gevent.spawn(wait)
thread1.join(timeout=timer)

# --

timer = Timeout.start_new(1)

thread2 = gevent.spawn(wait)
thread2.get(timeout=timer)

# --

gevent.with_timeout(1, wait)
</code>
</pre>

# Data Structures

## Events

Events are a form of asynchronous communication between
Greenlets.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult

a = AsyncResult()

def setter():
    """
    After 3 seconds set wake all threads waiting on the value of
    a.
    """
    gevent.sleep(3)
    a.set()

def waiter():
    """
    After 3 seconds the get call will unblock.
    """
    a.get() # blocking
    print 'I live!'

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

A extension of the Event object is the AsyncResult which
allows you to send a value along with the wakeup call. This is
sometimes called a future or a deferred, since it holds a 
reference to a future value that can be set on an arbitrary time
schedule.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')

def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print a.get()

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

## Queues

Queues are ordered sets of data that have the usual ``put`` / ``get``
operations but are written in a way such that they can be safely
manipulated across Greenlets.

For example if one Greenlet grabs an item off of the queue, the
same item will not grabbed by another Greenlet executing
simultaneously.

<pre>
<code class="python">import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print 'Worker %s got task %s' % (n, task)
        gevent.sleep(0.5)

    print 'Quitting time!'

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
</code>
</pre>

Queues can also block on either ``put`` or ``get`` as the need arises. 

Each of the ``put`` and ``get`` operations has a non-blocking
counterpart, ``put_nowait`` and 
``get_nowait`` which will not block, but instead raise
either ``gevent.queue.Empty`` or
``gevent.queue.Full`` in the operation is not possible.

In this example we have the boss running simultaneously to the
workers and have a restriction on the Queue that it can contain no
more than three elements. This restriction means that the ``put``
operation will block until there is space on the queue.
Conversely the ``get`` operation will block if there are
no elements on the queue to fetch, it also takes a timeout
argument to allow for the queue to exit with the exception
``gevent.queue.Empty`` if no work can found within the
time frame of the Timeout.

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(n):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print 'Worker %s got task %s' % (n, task)
            gevent.sleep(0.5)
    except Empty:
        print 'Quitting time!'

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print 'Assigned all work in iteration 1'

    for i in xrange(10,20):
        tasks.put(i)
    print 'Assigned all work in iteration 2'

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
</code>
</pre>

## Groups and Pools

## Locks and Semaphores

## Actors

The actor model is a higher level concurrency model popularized
by the language Erlang. In short the main idea is that you have a
collection of independent Actors which have an inbox from which
they receive messages from other Actors. The main loop inside the
Actor iterates through its messages and takes action according to
its desired behavior. 

Gevent does not have a primitive Actor type, but we can define
one very simply using a Queue inside of a subclassed Greenlet.

<pre>
<code class="python">import gevent

class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = queue.Queue()
        Greenlet.__init__(self)

    def recieve(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.recieve(message)

</code>
</pre>

In a use case:

<pre>
<code class="python">import gevent
from gevent.queue import Queue
from gevent import Greenlet

class Pinger(Actor):
    def recieve(self, message):
        print message
        pong.inbox.put('ping')
        gevent.sleep(0)

class Ponger(Actor):
    def recieve(self, message):
        print message
        ping.inbox.put('pong')
        gevent.sleep(0)

ping = Pinger()
pong = Ponger()

ping.start()
pong.start()

ping.inbox.put('start')
gevent.joinall([ping, pong])
</code>
</pre>

# Real World Applications

## Holding Side Effects

In this example we hold the side effects of executing an
arbitrary string, 

<pre>
<code class="python">from gevent import Greenlet

env = {}

def run_code(code, env={}):
    local = locals()
    local.update(env)
    exec(code, globals(), local)
    return local

while True:
    code = raw_input('>')

    g = Greenlet.spawn(run_code, code, env)
    g.join() # block until code executes

    # If succesfull then pass the locals to the next command
    if g.value:
        env = g.get()
    else:
        print g.exception
</code>
</pre> 


## WSGI Servers

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'
    body = 'Hello Cruel World!'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre> 

Performance on Gevent servers is phenomenal.

<pre>
<code class="shell">$ ab -n 10000 -c 100 http://127.0.0.1:8000/
</code>
</pre> 

## Long Polling

## Chat Server

The final motivating example, a realtime chat room. This example
requires <a href="http://flask.pocoo.org/">Flask</a> ( but not neccesarily so, you could use Django,
Pyramid, etc ). The corresponding Javascript and HTML files can
be found <a href="https://github.com/sdiehl/minichat">here</a>.


<pre>
<code class="python"># Micro gevent chatroom.
# ----------------------

from flask import Flask, render_template, request

from gevent import queue
from gevent.pywsgi import WSGIServer

import simplejson as json

app = Flask(__name__)
app.debug = True

class Room(object):

    def __init__(self):
        self.users = set()
        self.messages = []

    def backlog(self, size=25):
        return self.messages[-size:]

    def subscribe(self, user):
        self.users.add(user)

    def add(self, message):
        for user in self.users:
            print user
            user.queue.put_nowait(message)
        self.messages.append(message)

class User(object):

    def __init__(self):
        self.queue = queue.Queue()

rooms = {
    'foo': Room(),
    'bar': Room(),
}

users = {}

@app.route('/')
def choose_name():
    return render_template('choose.html')

@app.route('/&lt;uid&gt;')
def main(uid):
    return render_template('main.html',
        uid=uid,
        rooms=rooms.keys()
    )

@app.route('/&lt;room&gt;/&lt;uid&gt;')
def join(room, uid):
    user = users.get(uid, None)

    if not user:
        users[uid] = user = User()

    active_room = rooms[room]
    active_room.subscribe(user)
    print 'subscribe', active_room, user

    messages = active_room.backlog()

    return render_template('room.html',
        room=room, uid=uid, messages=messages)

@app.route("/put/&lt;room&gt;/&lt;uid&gt;", methods=["POST"])
def put(room, uid):
    user = users[uid]
    room = rooms[room]

    message = request.form['message']
    room.add(':'.join([uid, message]))

    return ''

@app.route("/poll/&lt;uid&gt;", methods=["POST"])
def poll(uid):
    try:
        msg = users[uid].queue.get(timeout=10)
    except queue.Empty:
        msg = []
    return json.dumps(msg)

if __name__ == "__main__":
    http = WSGIServer(('', 5000), app)
    http.serve_forever()
</code>
</pre>

## License

<p>
This is a collaborative document published under MIT license. Forking 
on <a href="https://github.com/sdiehl/gevent-tutorial">GitHub</a> is encouraged
</p>