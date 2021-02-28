---
title: "snap-pac moving to Python"
description: "I'm rewriting the main script from bash to Python for a few reason"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
image: images/python-powered.png
categories: [linux,bash,snapper,btrfs,python]
---

snap-pac is a set of pacman hooks and a bash script on Arch Linux that helps automate
pre/post snappper snapshotting. It's allows a user to easily rollback upgrades or any
pacman transaction using snapper.

For a few years now the script has been written in bash. That's been fine, but in my
opinion it makes it very difficult to add new features or even adequetly test the code.
Thus, I started transitioning the project to Python.

You can see the changes on the master branch in the repository. I've also add a Github
Workflow to automatically test the code every time master is updated either through a
push or a pull request.

# stdlib modules

The code itself doesn't require anything outside of the Python standard library. Some of
the stdlib modules that have made things really nice are argparse, configparser, and
pathlib. Below I describe a couple of nice things these add to the code.

Since this script needs to be called on both the pre and post snapshot via the pacman
hooks, I've made a command line argument to pass which snapshot type is intended, which
can either be "pre" or "post". Previously I had an if statement checking that the
argument is one of those two, but now with argparse I can simply add `choices` to
`add_argument`:

```python
parser = ArgumentParser(description="Script for taking pre/post snapper snapshots. Used with pacman hooks.")
parser.add_argument(dest="type", choices=["pre", "post"])
args = parser.parse_args()
```
No extra logic needs to be written to check the arguments.

Another nice feature is being able to add default arguments for the ini file. Here i add
a few defaults first and then read the ini file which can override them:

```python
config = ConfigParser()
config["DEFAULT"] = {
    "snapshot": False,
    "cleanup_algorithm": "number",
    "pre_description": parent_cmd,
    "post_description": packages,
    "desc_limit": 72
}
config["root"] = {
    "snapshot": True
}
config.read(ini_file)
```

Each snapper configuration has its own section in the ini file. If a snapper
configruation does not have a section I can just use the default settings by adding the
section myself:

```python
if snapper_config not in config:
    config.add_section(snapper_config)
```

With pathlib I can use the `read_text` and `write_text` methods to automatically open,
read/write, and then close the file. For example, this:

```python
with open(prefile, "w") as f:
    f.write(num)
```

can be replaced with this:
```python
prefile.write_text(num)
```

`prefile` is a `pathlib.Path` object.


# Object-oriented code

A second major advantage of python over bash is that I can write object-oriented code to
populate the snapper command. In the bash version I had to try to create the command
dynamically as a string. This made it difficult when the command itself has string
arguments that had to be quoted. Here's the Python class:

```python
class SnapperCmd:

    def __init__(self, config, snapshot_type, cleanup_algorithm, description="", nodbus=False, pre_number=None):
        self.cmd = ["snapper"]
        if nodbus:
            self.cmd.append("--no-dbus")
        self.cmd.append(f"--config {config} create")
        self.cmd.append(f"--type {snapshot_type}")
        self.cmd.append(f"--cleanup-algorithm {cleanup_algorithm}")
        self.cmd.append("--print-number")
        if description:
            self.cmd.append(f"--description \"{description}\"")
        if snapshot_type == "post":
            if pre_number is not None:
                self.cmd.append(f"--pre-number {pre_number}")
            else:
                raise ValueError("snapshot type specified as 'post' but no pre snapshot number passed.")

    def __call__(self):
        return os.popen(self.__str__()).read().rstrip("\n")

    def __str__(self):
        return " ".join(self.cmd)
```

After the object is initialized, the list containing the arguments is joined together as
a string and then run with `os.popen`.

# Tests with pytest

As I mentioned above, another advantage is that tests can actually be run. I'm not aware
of any simple way to write unit tests for a bash script. In my case I am using `pytest`
to run the tests in the Github workflow. Unfortunately I can't easily test the actual
snapshotting since that requires snapper and btrfs, but I can do that manually myself.

One note is that simply running `pytest` does not work correctly since the path of the
script is not added. Instead one must run `python -m pytest`.

In addition to using `pytest` I am using `flake8` to check code formatting
automatically.

# Installation

One thing I have not really changed is how the script is installed. It is a single
Python script that needs to reside under `/usr/share/libalpm/scripts`. I did look into
using `setup.py` to create a Python module but it does not seem worth the effort in
getting things right since it is a single Python file.

# Conclusion

Although I have the Python code in the master branch, I am doing further testing to
ensure that nothing breaks for users before making a release. Discussion of the code
changes is [in this issue](https://github.com/wesbarnett/snap-pac/issues/39).
