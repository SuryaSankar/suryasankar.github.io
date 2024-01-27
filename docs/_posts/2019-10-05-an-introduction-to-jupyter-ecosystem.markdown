---
layout: post
title:  "An introduction to Jupyter ecosystem"
date:   2019-10-05 00:26:37 +0530
categories: jupyter notebook
canonical_url: 'https://medium.com/techonometrics/an-introduction-to-the-jupyter-ecosystem-72005f7bdd78'
---

I had recently setup a Jupyterhub server for my company and wanted to blog about the same. Before we get into the setup instructions, it might be useful to have a brief overview of the Jupyter eco-system. When I started I was confused by the various terms which were being thrown around. So this post will be kind of a glossary of the various technologies which together constitute the Jupyter eco-system.

### First there was the shell
According to multiple surveys, Python is either the most popular or the second most popular programming language in the world. Frameworks like Django, Flask etc have made it a major contender in the server side application development landscape where it is second only to Javascript in the adoption rate. But the biggest success story of Python has been as the language of scientific computing and data processing. Python has become the go-to language for professionals from other domains for whom programming is only a part of their job description and would hence like a language which just gets the job done and gets out of the way. Python fit this requirement very well. The main reason for this was python’s readability, gentle learning curve and the community’s general preference for explicit code over implicit magic.

But there was another major factor as well. Python being an interpreted language, was one of the first major languages to offer a powerful interactive shell. The availability of an interactive shell is particularly important during the research and prototyping stage when you need to play around with data and see immediate results. Python shell allowed you to execute code blocks one at a time and view the results. Over years the need was felt for a richer and more interactive python shell and thus IPython shell was born.

Ipython shell offered many advantages over the built in python shell like tab completion, inspection of object internals, access to the underlying bash shell etc. You can check it out yourself by doing pip install ipython and then invoking ipython command. You will see the differences with the basic python shell.

The next major development that happened was Ipython notebooks. These were basically browser based ipython shells which could communicate with a python process running on the local machine or a remote server. Plus, the whole interactive session could be saved as “.ipynb” files which saved the inputs and outputs of all the cells in a json format, allowing the notebooks to be shared and re-executed by others. These ipython notebooks became very popular in the research community and in 2014, the notebook part of the ipython project was separated from the command line client and was spun off as a separate project — Jupyter

### Uses of Jupyter notebooks
If you are a research scientist or data analyst who is starting work on a new topic, one of the first things you are going to do is to open an ipython console and play around with the data you have. Once you are sure that you have come up with something that is worth sharing with your peers, you will want to convert the exploratory analysis you just did into a presentable format. Jupyter notebooks were developed to make this process very simple by just making the python console itself a shareable document.

Here are some examples,

1. A scientific paper on RNA sequence analysis published as a jupyter notebook
2. A notebook which has been created to explain Stoichometry for Chemical Engineering students

As you can see from above examples, the notebook is basically a collection of “Cells”, a cell being a block of code / text. Text and code blocks can be interspersed freely, so if you are exploring a mathematical formula for example, you can freely mix the textual annotations which explain the formula and pythonic code which can demonstrate the formula in action by passing some inputs to a method and retrieving the output. And the fact that these notebooks are built on top of web technologies, means that the jupyter notebook consoles are capable of supporting a much richer output than plain shell based consoles. Most visualization libraries these days provide out of the box support to render the output on a jupyter notebook. Here is an example notebook analysing US census data with some rich visualizations.

Researchers are not the only ones who can take advantage of the capabilities of Jupyter notebooks. Content writers can benefit as well. So a financial journalist reporting on the financials of a company can easily insert small snippets of code which pull the latest financial data of the company available on the internet and run some analysis on it and convert it into attractive visualizations. Similarly one can imagine how so many other kinds of content writers would benefit — technical writers writing posts on scientific topics which can be annotated with visualizations, journalists who want to report on some political or economic stats, the list is endless.

### How to install and use Jupyter notebooks
The easiest way to get started with Jupyter is to install the Anaconda environment on your local machine. It comes with Jupyter notebook bundled with it. Alternatively if you prefer pip as the package manager, you just install the python package as pip install jupyter and then you are ready to go. Calling jupyter notebook from the shell then spawns a server which can be interacted with via the web client.

### Nbconvert, Nbviewer
Since Jupyter notebooks have become a great hit with researchers and content creators, they also have to be convertable into various other presentation formats. So there is a command line tool available for the same — nbconvert using which Jupyter notebooks can be readily converted into a variety of presentation formats like HTML, PDF, Latex etc. The conversion is done simply by invoking the tool like this jupyter nbconvert --to pdf notebook.ipynb.

There is also a ready to deploy web server available called Nbviewer, which can be used to provide this Nbconvert as a service. So you can deploy Nbviewer on a server and allow users to submit links to jupyter notebooks and convert and render the generated html pages. There is a community maintained instance of nbviewer hosted here

### Integration with Blogging Engines/Static site generators
Converting notebooks into html is particularly interesting because this means that the notebooks can be considered as just another kind of a text format like rtf or markdown for writing blog posts, with the added advantage of supporting various dynamic functionalities like visualizations. When the notebooks are exported, these visualizations become a part of the exported html/pdf document.

Since nbconvert allows you to convert notebooks to html/markdown formats, you can use any major static site generator like Jekyll or Hugo, by just including the nbconvert command in your build process. There are tutorials available for the same like here and here

Another option which has arrived on the scene recently is Nikola . Nikola is a static site generator which includes support for ipython notebooks out of the box. If you are certain that most of your blogs will have some element of data analysis in them, using Nikola would be the straight forward option.

### JupyterLab
The default web interface that you see after installing Jupyter is the classic jupyter interface. But there is a next generation UI — which is called JupyterLab. It provides several features which were not available in the classic interface like built in support for viewing markdown documents, a launcher which can be used to launch new notebooks or a text document or code consoles. And Jupyterlab also has an excellent eco-system of third party extensions. In the next post, when we set up Jupyterhub, we will also set up the git extension for example.

### What is a JupyterHub?
A Jupyterhub is basically a web server which allows jupyter notebooks to be created and run by various users on a remote machine which can then be accessed via JupyterLab interfaces on the browser. Basically the browser console is pointed towards the remote Jupyterhub server instead of the local Jupyter server as would have been the case if you had just invoked jupyter or jupyter lab on the command line.

If you are using jupyter notebooks only for your own consumption, then the local server is sufficient for your purposes. But if you have to host the notebooks on a remote server and provide access to others, then it becomes necessary to setup a JupyterHub. The server which is spawned when launching a Jupyter notebook, does not support authentication. JupyterHub does. You can setup user accounts and allow access only to authorized users. This is essential if you are going to host the notebooks on a server Jupyterhub is typically used by data teams in companies and universities. But even an individual content creator/data analyst can benefit from setting up a Jupyterhub.

### Why Jupyterhub can serve as a good content management system for a technical blog
The Jupyter ecosystem has more than adequate tools necessary for a good content management system

The Jupyterlab interface which has a decent Markdown editor with previewing ability is good enough for writing textual posts
As already discussed above, the notebook format is excellent for technical literature, as it can let the author create original analyses with visualizations. The fact that it is in code means that the analyses and the visualizations can be updated easily. By contrast, if you use a traditional content management system like wordpress, you might have to modify the visualizations via some other third party tool and re-insert it in the post. By making this process painless, jupyter notebooks encourage multiple iterations on technical content, just like a programmer would write a program.
As discussed above, the notebooks can be easily converted into html files. And multiple static site generators like Jekyll, Hugo, Pelican, Nikola etc have tools which allow you to do this conversion as a part of the site building process itself. So we can easily serve the generated content via one of these tools. (We will discuss this in more detail in a later post)
Jupyterlab has an extension which provides a GUI for git integration. So, you can keep your content in a git repository. For a technical blog, having git as the version control system is a natural choice. Thus the data is stored separate from the content management system and the server. Even if the server were to crash unexpectedly, the data would still be safe in a git repository.
Jupyterhub supports various authentication mechanisms. Setting up a Google oauth authentication is very easy. So you can easily integrate it inside your company’s authentication system.
Python has several excellent visualization libraries like matplotlib, plotly, bokeh etc. Writing content as a jupyter notebook helps the author take advantage of these while creating visually appealing posts.

### How to set up Jupyterhub server
The simplest way to get a jupyterhub server up and running is to follow the instructions here — The Littlest Jupyterhub. The installation script provided there takes care of the entire setup process