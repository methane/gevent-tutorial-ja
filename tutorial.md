[TOC]

# はじめに

このチュートリアルはある程度の Python の知識を前提としていますが、
それ以上の知識は前提としていません。
並列プログラミングの知識も必要ありません。
このチュートリアルの目的は、 gevent を扱う道具を提供し、
読者がすでに持っている一般的な並列プログラミングの問題を手なづけて
非同期プログラムを書き始められるように手助けすることです。

### 寄稿者

時系列順の寄稿者:
[Stephen Diehl](http://www.stephendiehl.com)
[J&eacute;r&eacute;my Bethmont](https://github.com/jerem)
[sww](https://github.com/sww)
[Bruno Bigras](https://github.com/brunoqc)
[David Ripton](https://github.com/dripton)
[Travis Cline](https://github.com/traviscline)
[Boris Feld](https://github.com/Lothiraldan)
[youngsterxyf](https://github.com/youngsterxyf)
[Eddie Hebert](https://github.com/ehebert)
[Alexis Metaireau](http://notmyidea.org)
[Daniel Velkov](https://github.com/djv)

そして Denis Bilenko に、 gevent の開発とこのチュートリアルを作る上での
指導について感謝します。

この共同作業によるドキュメントは MIT ライセンスにて公開されています。
何か追記したいことがあったり、タイプミスを発見した場合は、
[Github](https://github.com/sdiehl/gevent-tutorial) で fork して pull
request を送ってください。その他のどんな貢献も歓迎します。

### 日本語訳について

時系列順の翻訳者:
[methane](http://github.com/methane)

翻訳は [gevent-tutorial-ja](http://github.com/methane/gevent-tutorial-ja)
で行なっています。
翻訳に対する修正依頼等はこちらにお願いします.


# コア

## Greenlets

gevent で使われている一番重要なパターンが <strong>Greenlet</strong> という
Python のC拡張の形で提供された軽量なコルーチンです。
すべての Greenlet はメインプログラムのOSプロセス内で実行されますが、
協調的にスケジューリングされます.

> Only one greenlet is ever running at any given time.
> (どの時点をとっても常にひとつの greenlet だけが動いている)

これは、 ``multiprocessing`` や ``threading`` が提供している、
OS がスケジューリングして本当に並列に動くプロセスや POSIX スレッドを利用した 
並列機構とは異なるものです。

## Synchronous & Asynchronous Execution

並行プログラムのコアとなる考え方は、大きいタスクは小さいサブタスクに分割して、
それらを1つずつ、あるいは *同期的に* 動かす代わりに、同時に、あるいは
*非同期に* 動かすことです。
2つのサブタスク間の切り替えは *コンテキストスイッチ* と呼ばれます。

gevent のコンテキストスイッチは *譲渡(yield)* によって行われます。
次の例では、2つのコンテキストが ``gevent.sleep(0)`` を実行することにより
お互いに譲渡しています。

[[[cog
import gevent

def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
]]]
[[[end]]]

次の制御フローを視覚化した画像を見るか、このサンプルプログラムを
デバッガーでステップ実行してコンテキストスイッチが起こる様子を確認してください。

![Greenlet Control Flow](flow.gif)

gevent が本当の力を発揮するのは、ネットワークやIOバウンドの、協調的に
スケジュールできる関数を実行するために利用する場合です。
gevent はすべての詳細部分の面倒を見て、可能な限りネットワークライブラリが
暗黙的に greenlet コンテキストを譲渡するようにします。
私はこれがどれだけ強力な方法なのかをうまく強調することができません。
でも、次の例が示してくれると思います。

この例では、 ``select()`` 関数は通常ブロックする関数で、複数の
ファイルディスクリプタをポーリングします。

[[[cog
import time
import gevent
from gevent import select

start = time.time()
tic = lambda: 'at %1.1f seconds' % (time.time() - start)

def gr1():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: ', tic())
    select.select([], [], [], 2)
    print('Ended Polling: ', tic())

def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: ', tic())
    select.select([], [], [], 2)
    print('Ended Polling: ', tic())

def gr3():
    print("Hey lets do some stuff while the greenlets poll, at", tic())
    gevent.sleep(1)

gevent.joinall([
    gevent.spawn(gr1),
    gevent.spawn(gr2),
    gevent.spawn(gr3),
])
]]]
[[[end]]]

もう一つのすこし人工的な例として、 *非決定論的な* (同じ入力に対して
出力が同じになるとは限らない) ``task`` 関数を定義しています。
この例では、 ``task`` 関数を実行すると副作用としてランダムな秒数だけ
タスクの実行を停止します。

[[[cog
import gevent
import random

def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0,2))
    print('Task', pid, 'done')

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
]]]
[[[end]]]

同期的に実行した場合、全てのタスクはシーケンシャルに実行され、
メインプログラムは各タスクを実行している間 *ブロックする*
(メインプログラムの実行が停止される) ことになります。


このプログラムの重要な部分は、与えられた関数を greenlet スレッドの中に
ラップする ``gevent.spawn`` です。
生成された greenlet のリストが ``threads`` 配列に格納され、
``gevent.joinall`` 関数に渡されます。この関数は今のプログラムを、与えられた
すべての greenlet を実行するためにブロックします。
すべての greenlet が終了した時に次のステップが実行されます。

注意すべき重要な点として、非同期に実行した場合の実行順序は本質的に
ランダムであり、トータルの実行時間は同期的に実行した場合よりもずっと
短くなります。実際、各タスクが2秒ずつ停止すると、同期的に実行した場合は
すべてのタスクを実行するのに20秒かかりますが、非同期に実行した場合は
各タスクが他のタスクの実行を止めないのでだいたい2秒で終了します。


もっと一般的なユースケースとして、サーバーからデータを非同期に取得します。
``fetch()`` の実行時間はリモートサーバーの負荷などによってリクエストごとに
異なります。

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

## Determinism

先に触れたように、 greenlet は決定論的に動作します。
同じように設定した greenlets があって、同じ入力が与えられた場合、
必ず同じ結果になります。例として、タスクを multiprocessing の pool に
与えた場合と gevent pool に与えた場合を比較します。

<pre>
<code class="python">
import time

def echo(i):
    time.sleep(0.001)
    return i

# Non Deterministic Process Pool

from multiprocessing.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print( run1 == run2 == run3 == run4 )

# Deterministic Gevent Pool

from gevent.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print( run1 == run2 == run3 == run4 )
</code>
</pre>

<pre>
<code class="python">False
True</code>
</pre>

gevent が通常決定論的だといっても、ソケットやファイルなどの外部の
サービスとのやりとりを始めると非決定論的な入力がプログラムに入り込んできます。
green スレッドが "決定論的な並行" であっても、 POSIX スレッドやプロセスを
使った場合に起きる問題の一部が起こります。

並行プログラミングにずっと付きまとう問題が
*レースコンディション(race condition)* です。
簡単に言うと、2つの並行するスレッドやプロセスが幾つかの共有リソース
(訳注:グローバル変数など)に依存しており、しかもそれを変更しようとする問題です。
結果としてそのリソースの状態は実行順序に依存してしまいます。
一般的にプログラマーはこの問題を可能な限り避けようとするべきです。
そうしないとプログラムのふるまいが全体として非決定性になってしまうからです。


レースコンディションを避ける一番の方法は、常に、全てのグローバルな状態を避ける事です。
グローバルな状態や import 時の副作用はいつでもあなたに噛みついてきます。
(訳注: import 時の副作用は、 import 順序によってプログラムの動作を変えてしまう、
Python でプログラミングをすると忘れた頃にハマる問題のタネです. 並行プログラミングとは
無関係です)

## Spawning Greenlets

gevent は greenlet の初期化のラッパーを幾つか提供しています。
幾つかのよくあるパターンは:

[[[cog
import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Each thread will be passed the message, and n arguments
    in its initialization.
    """
    gevent.sleep(n)
    print(message)

# Initialize a new Greenlet instance running the named function
# foo
thread1 = Greenlet.spawn(foo, "Hello", 1)

# Wrapper for creating and runing a new Greenlet from the named 
# function foo, with the passed arguments
thread2 = gevent.spawn(foo, "I live!", 2)

# Lambda expressions
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Block until all threads complete.
gevent.joinall(threads)
]]]
[[[end]]]

Greenlet クラスを直接使うだけでなく、 Greenlet のサブクラスを作って
``_run`` メソッドをオーバーライドすることもできます。

[[[cog
import gevent
from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)

g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
]]]
[[[end]]]


## Greenlet State

他のどんなコードでもあるように、 greenlet もいろいろな失敗をすることがあります。
greenlet は例外を投げそこねるかもしれませんし、停止できなくなったり、
システムリソースを食い過ぎるかもしれません。

greenlet の内部状態は基本的に時間とともに変化するパラメーターになります。
greenlet の状態をモニターするための幾つかのフラグがあります。

- ``started`` -- bool値: greenlet が開始しているかどうか.
- ``ready()`` -- bool値, greenlet が停止(halt)しているかどうか.
- ``successful()`` -- bool値, greenlet が例外を投げずに終了したかどうか.
- ``value`` -- 任意の値, greenlet が返した値.
- ``exception`` -- 例外, greenlet 内で投げられた例外.

[[[cog
import gevent

def win():
    return 'You win!'

def fail():
    raise Exception('You fail at failing.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started) # True
print(loser.started)  # True

# Exceptions raised in the Greenlet, stay inside the Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('This will never be reached')

print(winner.value) # 'You win!'
print(loser.value)  # None

print(winner.ready()) # True
print(loser.ready())  # True

print(winner.successful()) # True
print(loser.successful())  # False

# The exception raised in fail, will not propogate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.

print(loser.exception)

# It is possible though to raise the exception again outside
# raise loser.exception
# or with
# loser.get()
]]]
[[[end]]]

## Program Shutdown

メインプログラムが SIGQUIT を受信した時に yield しない greenlet
がプログラムを実行させ続ける可能性があります。
この状態は "ゾンビプロセス" と呼ばれ、 Python インタープリターの
外から kill してやる必要があります。

一般的なパターンはメインプログラムが SIGQUIT イベントを受信して、
exit する前に ``gevent.shutdown`` を呼び出すことです。

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

タイムアウトとは、コードブロックや greenlet の実行時間に対する制約です。

<pre>
<code class="python">
import gevent
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

もしくはコンテキストマネージャーとして ``with`` 文を使います。

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

加えて、 gevent は多くの greenlet やデータ構造に関する関数に timeout
引数を提供しています。例えば:

[[[cog
import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()
thread1 = gevent.spawn(wait)

try:
    thread1.join(timeout=timer)
except Timeout:
    print('Thread 1 timed out')

# --

timer = Timeout.start_new(1)
thread2 = gevent.spawn(wait)

try:
    thread2.get(timeout=timer)
except Timeout:
    print('Thread 2 timed out')

# --

try:
    gevent.with_timeout(1, wait)
except Timeout:
    print('Thread 3 timed out')

]]]
[[[end]]]

# データ構造

## Events

イベントは greenlet 間の非同期通信の一つです。

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

Event オブジェクトの拡張の AsyncResult を使うと、モーニングコール
(wakeup call) 付きの値を送ることができます。
これは将来どこかのタイミングで設定される値に対する参照を持っているので、
future とか deferred と呼ばれることもあります。

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

キューはデータの順序付き集合で、標準的な ``put / get`` 操作を持っています。
これは greenlet をまたいで安全に操作できるように実装されています。


例えば、ある greenlet がキューから要素を取得した時、同じ要素が並行して
動いている他の greenlet でも取得されることはありません。

[[[cog
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
]]]
[[[end]]]

キューは必要があれば ``put`` でも ``get`` でもブロックすることがあります。

``put`` と ``get`` にはそれぞれブロックしないバージョンとして
それぞれ ``put_nowait`` と ``get_nowait`` が用意されています。
これらの操作は実行できない場合はブロックする代わりに
``gevent.queue.Empty`` か ``gevent.queue.Full`` 例外を発生させます。

次の例では、並行して動いている boss と複数の worker がいて、
3要素以上格納できない制限付きのキューがあります。
この制限により、キューに空きが無いときは空きができるまで ``put``
操作がブロックして、キューが空の場合は要素が格納されるまで ``get``
操作がブロックします。
``get`` には timeout 引数を設定して、その時間内に要素を取得できない
場合は ``gevent.queue.Empty`` 例外を発生させて終了します。


[[[cog
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(n):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker %s got task %s' % (n, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print('Assigned all work in iteration 1')

    for i in xrange(10,20):
        tasks.put(i)
    print('Assigned all work in iteration 2')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
]]]
[[[end]]]

## Groups and Pools

グループとは、複数の greenlet をまとめてスケジュールしたり管理するものです。
Python の ``multiprocessing`` ライブラリの並列ディスパッチを置き換える用途にも使えます。

[[[cog
import gevent
from gevent.pool import Group

def talk(msg):
    for i in xrange(3):
        print(msg)

g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')
g3 = gevent.spawn(talk, 'fizz')

group = Group()
group.add(g1)
group.add(g2)
group.join()

group.add(g3)
group.join()
]]]
[[[end]]]

これは非同期なタスクのグループを管理するのに便利です。

上で述べたように、 Group はグループ化された greenlet に対してジョブを
ディスパッチし、その結果をいろいろな方法で取得するためのAPIを提供しています。

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Size of group', len(group))
    print('Hello from Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Ordered')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Unordered')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)

]]]
[[[end]]]

pool は並行数を制限しながら動的な数の greenlet を扱うためのものです。
たくさんのネットワークやIOバウンドのタスクを並列に実行したい場合に
最適です。

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Size of pool', len(pool))

pool.map(hello_from, xrange(3))
]]]
[[[end]]]

gevent を使ったサービスを作るときに、よく中央に pool を持った
構成で設計します。
例えばたくさんのソケットをポーリングするクラスです。

<pre>
<code class="python">from gevent.pool import Pool

class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()

</code>
</pre>

## Locks and Semaphores

セマフォ(Semaphore) は greenlet に並行アクセスや並行実行を調整する低レベルな
同期機構です。
セマフォは ``acquire`` と ``release`` というメソッドを提供しています。
``acquire`` と ``release`` の呼び出された回数のをセマフォの bound と呼びます。
bound が0になると、他の greenlet が ``release`` するまでブロックします。


[[[cog
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)

def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
]]]
[[[end]]]

bound が 1 のセマフォのことをロック(Lock)と言います。
ロックを使うと一つの greenlet だけを実行可能にすることができます。
プログラムのコンテキストでなにかのリソースを並行して複数の greenlet
から使わないようにするために利用されます。

## Thread Locals

Gevnet also allows you to specify data which is local the
greenlet context. Internally this is implemented as a global
lookup which addresses a private namespace keyed by the
greenlet's ``getcurrent()`` value.

[[[cog
import gevent
from gevent.local import local

stash = local()

def f1():
    stash.x = 1
    print(stash.x)

def f2():
    stash.y = 2
    print(stash.y)

    try:
        stash.x
    except AttributeError:
        print("x is not local to f2")

g1 = gevent.spawn(f1)
g2 = gevent.spawn(f2)

gevent.joinall([g1, g2])
]]]
[[[end]]]

Many web framework thats integrate with gevent store HTTP session
objects inside of gevent thread locals. For example using the
Werkzeug utility library and its proxy object we can create
Flask style request objects.

<pre>
<code class="python">from gevent.local import local
from werkzeug.local import LocalProxy
from werkzeug.wrappers import Request
from contextlib import contextmanager

from gevent.wsgi import WSGIServer

_requests = local()
request = LocalProxy(lambda: _requests.request)

@contextmanager
def sessionmanager(environ):
    _requests.request = Request(environ)
    yield
    _requests.request = None

def logic():
    return "Hello " + request.remote_addr

def application(environ, start_response):
    status = '200 OK'

    with sessionmanager(environ):
        body = logic()

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()


<code>
</pre>

Flask's system is more a bit sophisticated than this example, but the
idea of using thread locals as local session storage is nontheless the
same.

## Subprocess

As of Gevent 1.0, support has been added for cooperative waiting
on subprocess.

<pre>
<code class="python">import gevent
from gevent.subprocess import Popen, PIPE

# Uses a green pipe which is cooperative
sub = Popen(['uname'], stdout=PIPE)
read_output = gevent.spawn(sub.stdout.read)

output = read_output.join()
print(output.value)
<code>
</pre>

<pre>
<code class="python">Linux
<code>
</pre>

Many people also want to use gevent and multiprocessing together. This
can be done as most multiprocessing objects expose the underlying file
descriptors.

[[[cog
import gevent
from multiprocessing import Process, Pipe
from gevent.socket import wait_read, wait_write

# To Process
a, b = Pipe()

# From Process
c, d = Pipe()

def relay():
    for i in xrange(10):
        msg = b.recv()
        c.send(msg + " in " + str(i))

def put_msg():
    for i in xrange(10):
        wait_write(a.fileno())
        a.send('hi')

def get_msg():
    for i in xrange(10):
        wait_read(d.fileno())
        print(d.recv())

if __name__ == '__main__':
    proc = Process(target=relay)
    proc.start()

    g1 = gevent.spawn(get_msg)
    g2 = gevent.spawn(put_msg)
    gevent.joinall([g1, g2], timeout=1)
]]]
[[[end]]]

## Actors

アクターモデルとは Erlang 言語によって有名になった高レベルの並行プログラミングモデルです。
基本となる考え方を簡単に言うと、独立した複数のアクターが、他のアクターから
メッセージを受信するための受信箱を持っているというものです。
アクターの中のメインループは、メッセージを受信してはそれに対応する
行動を取ります。

gevent はプリミティブとしてのアクター型は持っていませんが、
Greenlet クラスを継承して Queue を使うことで簡単に実現できます。

<pre>
<code class="python">import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)

</code>
</pre>

利用例:

<pre>
<code class="python">import gevent
from gevent.queue import Queue
from gevent import Greenlet

class Pinger(Actor):
    def receive(self, message):
        print message
        pong.inbox.put('ping')
        gevent.sleep(0)

class Ponger(Actor):
    def receive(self, message):
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

## Gevent ZeroMQ

製作者によれば、 [ZeroMQ](http://www.zeromq.org/) は
"並行フレームワークのように振る舞うソケットライブラリ"
ということです。
並行・分散アプリケーションを作るときに、非常に強力な
メッセージングレイヤーになります。

ZeroMQ はたくさんの種類の socket プリミティブを提供しています。
一番シンプルなものは リクエスト-レスポンス ペアです。
この socket は ``send`` と ``recv`` というメソッドを持っていて、どちらも
通常はブロックします。しかし [Travis Cline](https://github.com/traviscline) が
gevent.socket を使って ZeroMQ socket をノンブロッキングにポーリングするように
してくれました。 ``pip install gevent-zeromq`` でインストールできます。

(訳注: これは pyzmq に取り込まれ、最新版の 2.2.0.1 では `zmq.green` を
import して利用することができます。)


[[[cog
# Note: Remember to ``pip install pyzmq gevent_zeromq``
import gevent
from gevent_zeromq import zmq

# Global Context
context = zmq.Context()

def server():
    server_socket = context.socket(zmq.REQ)
    server_socket.bind("tcp://127.0.0.1:5000")

    for request in range(1,10):
        server_socket.send("Hello")
        print('Switched to Server for ', request)
        # Implicit context switch occurs here
        server_socket.recv()

def client():
    client_socket = context.socket(zmq.REP)
    client_socket.connect("tcp://127.0.0.1:5000")

    for request in range(1,10):

        client_socket.recv()
        print('Switched to Client for ', request)
        # Implicit context switch occurs here
        client_socket.send("World")

publisher = gevent.spawn(server)
client    = gevent.spawn(client)

gevent.joinall([publisher, client])

]]]
[[[end]]]

## Simple Servers

<pre>
<code class="python">
# On Unix: Access with ``$ nc 127.0.0.1 5000`` 
# On Window: Access with ``$ telnet 127.0.0.1 5000`` 

from gevent.server import StreamServer

def handle(socket, address):
    socket.send("Hello from a telnet!\n")
    for i in range(5):
        socket.send(str(i) + '\n')
    socket.close()

server = StreamServer(('127.0.0.1', 5000), handle)
server.serve_forever()
</code>
</pre>

## WSGI Servers

gevent は HTTP のための2種類の WSGI サーバーを提供しています。
``wsgi`` と ``pywsgi`` です。

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer

gevent 1.0 より前のバージョンでは、 gevent は libev の代わりに libevent
を利用していました。 libevent は高速な HTTP サーバーを持っており、
gevent の `wsgi` サーバーで利用されていました。

gevent 1.0 からは libev が http サーバーを持っていないので、
``gevent.wsgi`` はピュアPythonで実装された ``gevent.pywsgi`` への
ただのエイリアスになっています。

## Streaming Servers

ストリーミングHTTPサービスを実現するには、まず HTTP ヘッダにコンテンツの
サイズを出力するのをやめます。その代わりに接続をつないだままにしておき、
データチャンクの先頭にそのチャンクの大きさを示す hex digit をつけて
送信します。サイズが0のチャンクを送るとストリームが終了します。


    HTTP/1.1 200 OK
    Content-Type: text/plain
    Transfer-Encoding: chunked

    8
    <p>Hello

    9
    World</p>

    0

pywsgi (gevent 1.0 以降は gevent.wsgi でも同じ) を使うと、ハンドラを
ジェネレータとして作成しチャンクを yield していくことでストリーミング
を実現できます。

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    yield "&lt;p&gt;Hello"
    yield "World&lt;/p&gt;"

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre> 

## Long Polling

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty
from gevent.pywsgi import WSGIServer
import simplejson as json

data_source = Queue()

def producer():
    while True:
        data_source.put_nowait('Hello World')
        gevent.sleep(1)

def ajax_endpoint(environ, start_response):
    status = '200 OK'
    headers = [
        ('Content-Type', 'application/json')
    ]

    start_response(status, headers)

    while True:
        try:
            datum = data_source.get(timeout=5)
            yield json.dumps(datum) + '\n'
        except Empty:
            pass


gevent.spawn(producer)

WSGIServer(('', 8000), ajax_endpoint).serve_forever()

</code>
</pre>

## Websockets

Websocket のサンプルは [gevent-websocket](https://bitbucket.org/Jeffrey/gevent-websocket/src)
を利用しています。

<pre>
<code class="python"># Simple gevent-websocket server
import json
import random

from gevent import pywsgi, sleep
from geventwebsocket.handler import WebSocketHandler

class WebSocketApp(object):
    '''Send random data to the websocket'''

    def __call__(self, environ, start_response):
        ws = environ['wsgi.websocket']
        x = 0
        while True:
            data = json.dumps({'x': x, 'y': random.randint(1, 5)})
            ws.send(data)
            x += 1
            sleep(0.5)

server = pywsgi.WSGIServer(("", 10000), WebSocketApp(),
    handler_class=WebSocketHandler)
server.serve_forever()
</code>
</pre>

HTML Page:

    <html>
        <head>
            <title>Minimal websocket application</title>
            <script type="text/javascript" src="jquery.min.js"></script>
            <script type="text/javascript">
            $(function() {
                // Open up a connection to our server
                var ws = new WebSocket("ws://localhost:10000/");

                // What do we do when we get a message?
                ws.onmessage = function(evt) {
                    $("#placeholder").append('<p>' + evt.data + '</p>')
                }
                // Just update our conn_status field with the connection status
                ws.onopen = function(evt) {
                    $('#conn_status').html('<b>Connected</b>');
                }
                ws.onerror = function(evt) {
                    $('#conn_status').html('<b>Error</b>');
                }
                ws.onclose = function(evt) {
                    $('#conn_status').html('<b>Closed</b>');
                }
            });
        </script>
        </head>
        <body>
            <h1>WebSocket Example</h1>
            <div id="conn_status">Not Connected</div>
            <div id="placeholder" style="width:600px;height:300px;"></div>
        </body>
    </html>


## Chat Server

最後に意欲的なサンプルとして、リアルタイムチャットルームを作ります。
このサンプルは [Flask](http://flask.pocoo.org/) を利用しています。
(代わりに Django や Pyramid を使ってもいいですよ！)
必要な JavaScript と HTML ファイルは [ここ](https://github.com/sdiehl/minichat)
にあります。


<pre>
<code class="python"># Micro gevent chatroom.
# ----------------------

from flask import Flask, render_template, request

from gevent import queue
from gevent.pywsgi import WSGIServer

import simplejson as json

app = Flask(__name__)
app.debug = True

rooms = {
    'topic1': Room(),
    'topic2': Room(),
}

users = {}

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
