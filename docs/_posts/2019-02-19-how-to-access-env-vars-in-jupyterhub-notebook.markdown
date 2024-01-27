---
layout: post
title:  "How to access environment variables in a notebook on Jupyterhub server"
date:   2019-02-19 00:26:37 +0530
categories: jupyter notebook
canonical_url: 'https://medium.com/techonometrics/how-to-access-environment-variables-in-a-notebook-on-a-jupyterhub-server-c9f53e256bc9'
---

Recently I had to create some jupyter notebooks to analyse a database. Since the database credentials were not meant to be committed as part of the code, I decided to keep them as environment variables. And it turned out to be surprisingly tricky to access the environment variables in the notebook. The first complication was that it was a Jupyterhub server and not a single instance notebook. When you launch a jupyter notebook directly from the command line, it has access to the environment variables defined in the .bashrc startup scripts. But that is not the case here. We will typically run Jupyterhub as a daemon and the Environment variables need to be passed in via that daemon script. And then the config file we use for the hub must also know to load those variables when we launch a new notebook. After much trial and error and googling, the following steps worked.

### Step 1 - Add the environment variables to the daemon scripts
Your jupyterhub startup script will typically reside at this place - `/lib/systemd/system/jupyterhub.service`
It should look like this

```
[Unit] 
Description=Jupyterhub 
[Service] 
User=root 
ExecStart=/usr/local/bin/jupyterhub -f /home/username/jupyterhub_repo/jupyterhub_config.py 
[Install] 
WantedBy=multi-user.target
```

Lets now add the desired environment variable in this file

```
[Unit] 
Description=Jupyterhub
[Service] 
User=root
Environment=GOOGLE_API_KEY='somesecretkeystring' Environment=GITHUB_API_KEY='anothersecretkeystring'
ExecStart=/usr/local/bin/jupyterhub -f /home/username/jupyterhub_repo/jupyterhub_config.py
[Install] 
WantedBy=multi-user.target
```

Note that we are adding 2 directives The first directive Environment is used to initialize an environment variable. We can initialize as many variables as we want in the above manner. But this step in itself is not enough. You will find that you still cannot access the variable inside your jupyterhub notebook shells. The reason is that the above directive only passes the environment variables to the command mentioned in the ExecStart directive. You need to further tell the systemctl daemon to pass the variables to the created process as well.

### Step 2 - Passing the variables to the Jupyterhub process
We use the PassEnvironment directive to achieve this

```
[Unit] 
Description=Jupyterhub
[Service] 
User=root
Environment=GOOGLE_API_KEY='somesecretkeystring' Environment=GITHUB_API_KEY='anothersecretkeystring'
PassEnvironment=GOOGLE_API_KEY GITHUB_API_KEY
ExecStart=/usr/local/bin/jupyterhub -f /home/username/jupyterhub_repo/jupyterhub_config.py
[Install] 
WantedBy=multi-user.target
```

Now the variables have been passed to the Jupyterhub process. But there is still one more step left

### Step 3 - Passing the variables to the ipython shell
While the above step passed the variables to the Jupyterhub process, you will find that these are not yet available in your Ipython shell launched inside Jupyterhub. This needs to be done by making Jupyterhub pass these variables down when it spawns a new shell. We do this by modifying the jupyterhub_config.py that we use to configure our Jupyterhub instance. Note that the location of this file is already mentioned in the ExecStart command in the daemon script. You would usually place this file in the root of your jupyterhub repository
To pass the variables, add the following lines to your config file

```
import os
for var in os.environ:
  c.Spawner.env_keep.append(var)
```

This passes on all environment variables to the spawned shell. Alternatively, you can also choose to pass only specific variables in the above script.
With these steps, you can now start using the environment variables inside jupyterhub. There are some obvious limitations to this approach. We are being forced to define the environment variables inside the daemon script - which means we might have to duplicate the values which were already defined in the .bashrc startup script. But I think the main advantage is that environment variables are easily usable by everyone and when your notebook uses code like os.environ["GOOGLE_API_KEY"], it makes it easy for other checking out the notebook to directly use it for their purpose by just changing the environment variables in their environment.