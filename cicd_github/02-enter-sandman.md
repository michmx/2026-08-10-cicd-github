# Exit Codes

:::{admonition} Overview
:class: note
**Teaching:** 10 min

**Questions**
- What is an exit code?

**Objectives**
- Understand exit codes
- How to print exit codes
- How to set exit codes in a script
- How to ignore exit codes
- Create a script that terminates in success/error
:::

As we enter the first episode of the Continuous Integration / Continuous Deployment (CI/CD) session, we learn how to exit.

## Start by Exiting

How does a general task know whether or not a script finished correctly or not? You could parse (`grep`) the output:

```bash
> ls nonexistent-file
```

```text
ls: cannot access 'nonexistent-file': No such file or directory
```

But every command outputs something differently. Instead, scripts also have an (invisible) exit code:

```bash
> ls nonexistent-file
> echo $?
```

```text
ls: cannot access 'nonexistent-file': No such file or directory
2
```

The exit code is `2` indicating failure. What about on success? The exit code is `0` like so:

```bash
> echo
> echo $?
```

```text
0
```

But this works for any command you run on the command line! For example, if I mistyped `git status`:

```bash
> git stauts
> echo $?
```

```text
git: 'stauts' is not a git command. See 'git --help'.

The most similar command is
  status
1
```

and there, the exit code is non-zero -- a failure.

:::{admonition} Exit Code is not a Boolean
:class: tip
You've probably trained your intuition to think of `0` as false. However, exit code of `0` means there was no error. If you feel queasy about remembering this, imagine that the question asked is "Was there an error in executing the command?" `0` means "no" and non-zero (`1`, `2`, ...) means "yes".
:::

Try out some other commands on your system, and see what things look like.

## Printing Exit Codes

As you've seen above, the exit code from the last executed command is stored in the `$?` environment variable. Accessing from a shell is easy `echo $?`. What about from python? There are many different ways depending on which library you use. Using similar examples above, we can use the `getstatusoutput()` call:


```python
>>> from subprocess import getstatusoutput
>>> status,output=getstatusoutput('ls')
>>> status
0
>>> status,output=getstatusoutput('ls nonexistent-file')
>>> status
2
```

Once inside the Python interpreter, simply type `exit()` then press enter, to exit. It may happen that this returns a different exit code than from the command line (indicating there's some internal implementation in Python). All you need to be concerned with is that the exit code was non-zero (there was an error).

## Setting Exit Codes

So now that we can get those exit codes, how can we set them? Let's explore this in `shell` and in `python3`.

### Shell

Create a file called `validate_energy.sh` with the following content:

```bash
#!/usr/bin/env bash

if [ $1 -lt 0 ]
then
  echo "Energy must be positive"
  exit 59
else
  exit 0
fi
```

and then make it executable `chmod +x validate_energy.sh`. Now, try running it with `./validate_energy.sh -10` and `./validate_energy.sh 10` and see what those exit codes are with `echo $?`.

### Python

The same can be done in a python file. Create a file called `validate_energy.py` with the following content:

```python
#!/usr/bin/env python3

import sys

if int(sys.argv[1]) < 0:
  print("Energy must be positive")
  sys.exit(59)
else:
  sys.exit(0)
```

and then make it executable `chmod +x validate_energy.py`. Now, try running it with `./validate_energy.py -10` and `./validate_energy.py 10` and see what those exit codes are. Déjà vu?


## Assert

An assertion is a sanity-check carried out by the `assert` statement, useful when testing or debugging code.

Let's create a file called `energy_assert.py` with the following content:
```python
#!/usr/bin/env python3

import sys

energy = int(sys.argv[1]) 
assert energy > 0, "Energy must be positive"
```

and then run it with `python3 energy_assert.py -10`.

What happens when an assertion fails in python?

```text
Traceback (most recent call last):
  File "energy_assert.py", line 3, in <module>
    assert energy > 0, "Energy must be positive"
           ^^^^^^^^^^
AssertionError: Energy must be positive
```

An exception is raised, `AssertionError`. The nice thing about python is that all unhandled exceptions return a non-zero exit code:
```bash
> echo $?
1
```

We can see that assertions automatically indicate failure in a script.


:::{admonition} Key Points
:class: note
- Exit codes are used to identify if a command or script executed with errors or not
- Assertions automatically indicate failure in a script
:::
