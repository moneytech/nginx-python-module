*******************
Nginx Python Module
*******************

The module allows using Python in nginx both at configuration stage and in
runtime.


Compatibility
=============

- nginx version >= 1.11.5 (HTTP-only version can be compiled with 1.11.2)
- Python version: 2.7
- tested on recent Linux, FreeBSD and MacOS


Build
=====

Configuring nginx with the module::

    # static module
    $ ./configure --add-module=/path/to/nginx-python-module

    # dynamic module
    $ ./configure --add-dynamic-module=/path/to/nginx-python-module

    # sync-only version (no blocking operations substitution)
    $ ./configure --add-module=/path/to/nginx-python-module
                  --with-cc-opt=-DNGX_PYTHON_SYNC=1

    # a specific Python installation can be used by exporting
    # the path to python-config prior to configuring
    $ export PYTHON_CONFIG=/path/to/python-config


Tests
=====

Like in standard nginx tests, the following environment variables are supported

- ``TEST_NGINX_BINARY`` - path to the nginx binary
- ``TEST_NGINX_CATLOG`` - dump error.log to stderr
- ``TEST_NGINX_LEAVE`` - do not remove test directory

Running tests::

    # run all tests
    $ python t
    
    # add verbosity with -v, get help with -h
    $ python t -v

    # run an individual test
    $ python t/test_http_basic.py


Directives
==========


Global Scope
------------

- ``python`` - execute Python code in config time
- ``python_include`` - include and execute Python code in config time
- ``python_stack_size`` - set stack size for unblocked code, default is 32k

HTTP Scope
----------

- ``python`` - execute Python code in config time
- ``python_include`` - include and execute Python code in config time
- ``python_set`` - create Python variable (one-line)
- ``python_access`` - set up Python access handler (one-line, blocking ops)
- ``python_log`` - set up Python log handler (one-line)
- ``python_content`` - set up Python location content handler (one-line,
  blocking ops)

Stream Scope
------------

- ``python`` - execute Python code in config time
- ``python_include`` - include and execute Python code in config time
- ``python_set`` - create Python variable (one-line)
- ``python_access`` - set up Python access handler (one-line, blocking ops)
- ``python_preread`` - set up Python access handler (one-line, blocking ops)
- ``python_log`` - set up Python log handler (one-line)
- ``python_content`` - set up Python server content handler (one-line,
  blocking ops)


Objects and namespaces
======================

default namespaces
------------------

In HTTP default namespace the current HTTP request instance ``r`` is available.

- ``hi{}`` - input headers (readonly)
- ``ho{}`` - output headers (read-write)
- ``var{}`` - nginx variables (readonly)
- ``arg{}`` - nginx arguments (readonly)
- ``ctx{}`` - request dictionary (read-write)
- ``status`` - HTTP status (read-write)
- ``log(msg, level)`` - write a message to nginx error log with given level
- ``sendHeader()`` - send HTTP header to client
- ``send(data, flags)`` - send a piece of output body, optional flags are
  ``SEND_LAST`` and ``SEND_FLUSH``

In Stream default namespace the current Stream session instance ``s`` is
available.

- ``buf`` - preread/UDP buffer (readonly)
- ``sock`` - client socket, I/O is allowed only at content phase
- ``var{}`` - nginx variables (readonly)
- ``ctx{}`` - session dictionary (read-write)
- ``log(msg, level)`` - write a message to nginx error log with given level

ngx namespace
-------------

In this namespace, standard constants are available

Standard nginx result codes

- ``OK``
- ``ERROR``
- ``AGAIN``
- ``BUSY``
- ``DONE``
- ``DECLINED``
- ``ABORT``

Log error levels

- ``LOG_EMERG``
- ``LOG_ALERT``
- ``LOG_CRIT``
- ``LOG_ERR``
- ``LOG_WARN``
- ``LOG_NOTICE``
- ``LOG_INFO``
- ``LOG_DEBUG``

Send flags

- ``SEND_FLUSH``
- ``SEND_LAST``


Blocking operations
===================

Nginx is a non-blocking server.  Using blocking operations while serving client
requests, will significantly decrease its performance.  The nginx-python-module
provides unblocked substitutions for common blocking operations in Python, and
makes these changes transparent for user.  This means, you can use common
blocking Python operations, while their implementations will rely on nginx
non-blocking core.  The list of classes and functions unblocked by the module:

- ``socket.socket`` class.  Unconnected (UDP) sockets, as well as Python SSL
  socket wrappers are not supported.
- ``socket.gethostbyname()`` and other resolve functions.  The ``resolver``
  directive in the current location is required for these functions.
- ``time.sleep()`` function.


Default Python namespace
========================

For each nginx configuration a new default Python namespace is created.  This
namespace is shared among all global, HTTP or Stream scopes in configuration
time, as well as HTTP requests and Stream sessions in runtime.  The namespace
can be initialized with the ``python`` and ``python_include`` directives, which
operate at configuration time.


Examples
========

Remote configuration
--------------------

Loading the essential part of nginx configuration file from a remote server::

    # nginx.conf

    python 'import urllib';
    python 'urllib.URLopener().retrieve("http://127.0.0.1:8888/nginx.conf", "/tmp/nginx.conf")';

    include /tmp/nginx.conf;

Variables
---------
::

    # nginx.conf

    events {}

    http {
        python "import hashlib";

        # md5($arg_foo)
        python_set $md5 "hashlib.md5(r.arg['foo']).hexdigest()";

        server {
            listen 8000;
            location / {
                return 200 $md5;
            }
        }
    }

Phase handlers
--------------

Dynamic Python module is used in this example::

    # nginx.conf

    load_module modules/ngx_python_module.so;

    events {}

    http {
        python_include inc.py;
        python_access "access(r)";

        server {
            listen 8000;
            location / {
                python_content "content(r)";
            }
        }
    }


    # inc.py

    import ngx
    import time
    import socket

    def access(r):
        r.log('access phase', ngx.LOG_INFO)
        r.ctx['code'] = 221

        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(('127.0.0.1', 8001))
        s.settimeout(2)
        s.send('foo')
        r.ho['X-Out'] = s.recv(10)

    def content(r):
        r.status = r.ctx['code']
        r.sendHeader()
        r.send('1234567890');
        r.send('abcdefgefg', ngx.SEND_LAST)

UDP socket
----------
::

    # nginx.conf

    events {}

    http {
        python_include inc.py;
        python_access "access(r)";

        server {
            listen 8000;
            location / {
                root html;
            }
        }
    }


    # inc.py

    import socket

    # send each $request via UDP to 127.0.0.1:6000

    ds = None

    def access(r):
        global ds

        if ds is None:
            ds = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            ds.connect(('127.0.0.1', 6000))

        ds.send(r.var['request'])

HTTP request in runtime
-----------------------
::

    # nginx.conf

    events {}

    http {
        python_include inc.py;
        python_access "access(r)";

        server {
            listen 8000;
            location / {
                root html;
            }
        }

        server {
            listen 8001;
            location / {
                return 200 foo;
            }
        }
    }


    # inc.py

    import httplib

    def access(r):
        conn = httplib.HTTPConnection("127.0.0.1", 8001)
        conn.request('GET', '/')
        resp = conn.getresponse()

        r.ho['x-status'] = resp.status;
        r.ho['x-reason'] = resp.reason;
        r.ho['x-body'] = resp.read()

Echo server
-----------
::

    # nginx.conf

    events {}

    stream {
        python_include inc.py;

        server {
            listen 8000;
            python_content echo(s);
        }
    }


    # inc.py

    def echo(s):
        while True:
            b = s.sock.recv(128)
            if len(b) == 0:
                return
            s.sock.sendall(b)
