---
title: "Create a venv for a Jupyter kernel"
description: "How to quickly setup a virtual environment for use as a Jupyter kernel"
layout: post
toc: false
comments: true
image: images/venv.png
hide: false
search_exclude: false
categories: [deeplearning]
---

After I setup my deep learning box, I installed JupyterHub following [these
instructions](https://jupyterhub.readthedocs.io/en/stable/installation-guide-hard.html#part-1-jupyterhub-and-jupyterlab).
I skipped the conda part myself since I like to run things in Python `venv`'s (virtual
environments). 

One aspect that I struggled with in finding in the documentation was how to setup a
Python `venv` to be used for a Jupyter kernel. It was actually much simpler than
expected.

First, create a `venv` that will be used for the kernel:

```bash
python -m venv my-venv
```

Then activate that virtual environment:

```bash
source ./my-venv/bin/activate
```

After that install ipykernel:

```bash
python -m pip install ipykernel
```

Lastly, register that environment as a kernel as described
[here](https://ipython.readthedocs.io/en/stable/install/kernel_install.html#kernels-for-different-environments):


```bash
python -m ipykernel install --user --name my-venv --display-name "Python (my-venv)"
```

That's it! Now when you log into your JupyterHub and start a notebook, you should see it
as available. 

## Installing packages

Whenenever you want to install new packages to be available in your kernel
simply install them to your virtual environment. You can ssh into your box, activate
your venv, and run pip to install them.

Alternatively, you can use the `%pip` magic in Jupyter to run the installation. For
example, to install pandas:

```bash
%pip install pandas
```

Note that this will give you different results from running `!pip`, which is running it
as a shell command. That won't use the pip in the virtual environment being utilized by
the kernel like `%pip` will. Neither will `!python -m pip`. You will end up installing
your packages in the wrong location if you use the shell form.

![]({{ site.baseurl }}/images/venv.png "Our virtual environment is now available as a
kernel.")
