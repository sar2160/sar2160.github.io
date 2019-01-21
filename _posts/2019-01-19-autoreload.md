---
layout: post
title: "The Authoritative 10xers Python Data Science Workflow "
categories: [data]
tags: [python, data-science, IDE]
---

One way to find a voice in the data science community is to proclaim _your_ workflow is the best and that all others are total trash, suitable for 4.5Xers at most, not a 10Xer like yourself! I kid, but also I think I've got a workflow that just ...ummm.. _works_ figured out, and why not share it. If you don't already know it, let me introduce you to the Jupyter extension `%load_ext autoreload`.

Some people like Jupyter Notebooks, some people [like using a more traditional IDE](https://www.reddit.com/r/Python/comments/9aoi35/i_dont_like_notebooks_joel_grus_jupytercon_2018/), and hey, there are points for both; IMO `%load_ext autoreload` is a way to harness the better aspects of each. All the extension really does is automatically reload your python imports every couple seconds. But what this allows you to do is code `.py` functions  in your IDE of choice (I like [VSCode](https://code.visualstudio.com)), import them into your Jupyter Notebook, and then immediately have them accessible for use. It's the best of both worlds!

## Benefits:
* **Organization**: keep your supporting code structured in a project folder with `models.py`, `preprocess.py`, etc..  It's there if someone wants to see it. 
* **Readability**: Your Jupyter Notebooks become the project output/summary that most people want to read, instead of an endless slog of cells to execute and scroll through. 
* **Scalability**: Your underlying code can grow as the need arises, instead of adding more and more stuff to a notebook, or creating poorly documented "Project-1-Copy.ipynb" files whenever you want to try something new. This folder structure already approximates the structure of a real python package, so you can even grow a project into a package without needless refactoring. 
* **Version Control**: Using `git` with predominantly `.py` files lets you track the code you care about, instead of all the embedded `html`  cruft in Jupyter Notebooks. 

## Drawbacks
* Jupyter Notebooks still have "execute-cells-out-of-order-and-get-weird-results" issue people don't like (and I don't either). But in this workflow the Notebook should only contain the important stuff like results and interpretation, so ideally you never are tempted to execute stuff out of order just to try something. All that work should be done in `.py` files.
* `.ipynb` files still don't play that well with version control, but keeping them to a minimum also helps.

I used this kind of structure in this [neural network project](https://github.com/sar2160/ecbm4040-finalproject) if you want to see an example. And here is the summary notebook:  

<iframe src="https://nbviewer.jupyter.org/github/sar2160/ecbm4040-finalproject/blob/master/CNN%20Project-Training.ipynb" width="100%" height="800" scrolling="yes"></iframe>  




**Note**: There's some cool IDE extensions like [Neuron](https://marketplace.visualstudio.com/items?itemName=neuron.neuron-IPE) for VSCode that try to bring the Notebook experience into the IDE, which is another way to solve this problem. At the moment I just don't find them as reliable or as fluid to work with as a simple `ctrl-tab` back and forth between pure a text editor and a Jupyter Notebook. This gap is probably going to shrink as these extensions get further developed. 