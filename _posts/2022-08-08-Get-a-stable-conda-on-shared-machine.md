---
layout: post
title: "How to get a stable conda environment on a shared, administrated machine"
date: 2022-08-08 19:00:00 +0200
categories: jekyll update
---

*disclaimer: do not annoy your system admin! If you are in doubt, check with your system admin that they are ok with you cloning their env and modifying it!*

## A few words about conda environments on shared machines

Conda environments are a convenient and common way to manage software. This post is not an introduction to ```conda``` (google will point you to many good resources on the topic) but, basically, ```conda``` allows you to easily switch between different software environments, ie sets of packages in given versions, to manage and keep track of software versions, and to keep different environments side by side with packages in versions that would otherwise conflict. All of it, without the need for admin rights on the host machine once conda is installed. Conda environments can manage not only python packages (though they are often used for this, possibly together with pip), but also "normal" packages: for example, ```PostgreSQL``` can be installed in a conda environment with a ```conda install postgresql``` command (note that, unlike a ```sudo apt install postgresql``` command, no admin rights are needed). Many libraries, programs, and software tools are packaged for conda, and if you need some software that is not packaged for conda yet, you can always ask the maintainers if they could distribute it via conda.

It is quite common that admins on shared machines (such as shared servers, or machines used for heavy computations) use ```conda``` for offering to their users a set of packages that are configured correctly to run on the machine. For example, a conda environment can provide support for GPUs, include libraries that are finicky to install, etc. This is very convenient for users, who can simply ```conda activate``` the environment provided by their admin and be ready to work. However, this comes with a couple of drawbacks: i) if a user wants to use an additional package, the user will typically need to ask the admin to install it in their favorite conda environment. This may take time if the admin is busy. ii) If the admin decides, for one or another reason, to update a conda environment, or "retire" it and remove it, users may end up not being able to run their programs as they used to.

A workaround as a user who has access to their own writeable folders is to locally clone the conda environment created by the admin (which is set up to be adapted to the shared machine hardware and characteristics, which may be quite a bit of work to get right), and then use this clone of the environment as their working environment for future work. This cloned environment i) can be tuned by the user (who can install additional packages as they like), ii) will not go away or get modified by anybody else than the user (as long as the user is the only one who has write rights on the folder chosen). (Note: of course, if you feel adventurous, you could also set up a new conda environment from scratch in a writeable folder, but that may be quite a bit of work if there are tricky dependencies.)

## Making sure ```conda``` is available

Let us assume first that ```conda``` is available. You can check that this is the case by issuing a command:

```bash
$ conda --version
conda 4.12.0
```

If you get an error message or similar, this means that you do not have access to ```conda```; check how to get access to ```conda``` on your specific machine! For example, on my machine, I needed to issue at the start of my session a command in the kind:

```bash
$ source SOME_SYSTEM_PATH/conda.sh
```

On your machine, you may need to either ```source``` a specific path, or ```module load``` a system module, or else to get access to ```conda```.

## Finding the conda environment you want to clone as a base for your own environment

Once you have access to ```conda```, you can easily list the conda envs available. For example on my server it looks something like:

```bash
$ conda info --envs
# conda environments:
#
base                  *  SOME_BASE_PATH/install
SOME_ENV                 SOME_BASE_PATH/envs/SOME_ENV
```

This means that the conda environment SOME\_ENV can be activated by:

```bash
$ conda activate SOME_ENV
(SOME_ENV) $ 
```

at which point you will be using the software made available by SOME\_ENV, and deactivated by:

```bash
(SOME_ENV) $ conda deactivate
$ 
```

## Cloning the environment into your own directory

At this stage, you are ready to clone the conda environment prepared by your system admin into your own folder to get your own cloned environment. For this, let's assume that you have access to a path SOME_USER_PATH where you have read and write access, and quite a lot of space (since conda environments can quickly add up and hold quite some space on the filesystem).

To create your clone environment, all you need to do is:

```bash
$ conda create --prefix=SOME_USER_PATH/ENVNAME --clone SOME_BASE_PATH/envs/SOME_ENV
```

The command may take a few minutes to run (conda environments can be quite heavy), but, once the command has run fully, you will now have an exact copy of the SOME\_ENV conda environment, except, in a directory under your control, so you can do whatever you want with your conda environment clone.

## Activating and modifying your own cloned environment

From there, you can use your conda environment clone exactly as you want, for example activating it, adding some packages to it, using it to run some code, and deactivating it:

```bash
$ conda activate SOME_USER_PATH/ENVNAME
(ENVNAME) $ conda install SOME_PACKAGE
(ENVNAME) $ RUN_SOME_CODE
(ENVNAME) $ conda deactivate
$ 
```

## Alternative scenario: create your own conda env from scratch

As a note, I assumed that you wanted to base yourself on the conda environment of your admin in order to build up on their hard work (such as, providing toolchains that work well on the specific kind of underlying server used), but of course you can also create your own conda environment from scratch, by not providing a clone argument:

```bash
$ conda create --prefix=SOME_USER_PATH/ENVNAME
```

In this case, you will get an environment with only the "default" minimal content in it.

## Tips: making your life easier with .condarc

Conda supports configuration and customization through the ```.condarc``` file. To create it if it does not already exist, simply type ```conda config```, or create a ```~/.condarc``` file. You can edit this file to contain, for example, the path to your own custom conda environments, if these are scattered a bit around. Note that there may be already a ```.condarc``` set up by your admin when activating conda on your platform, but the good thing is, if you write your own ```~/.condarc```, conda will merge the information it contains with the information present on the "admin set" ```.condarc``` configuration.

So, to make conda aware of your custom conda environments at locations under your control, in addition to the admin provided conda environments, you can simply edit your ```~/.condarc``` to contain:

```
envs_dirs:
  - PATH_TO_THE_LOCATION_OF_YOUR_ENVS
```

The ```.condarc``` allows you to customize many more aspects of conda; google for more information!

## Tips: automatically activate / deactivate conda environments with direnv

The ```direnv``` command can be used to automatically activate / deactivate conda environment when you navigate into a directory. That can be very convenient for automating workflows, and documenting which conda env should be used for each project. Here is a short summary of how to set up and use ```direnv``` (google for more information, as usual). This will assume that you use bash, there will be some small changes with other shells.

- install ```direnv```; this requires both a ```sudo apt install direnv```, and adding a command to activate direnv at the end of your ```.bashrc```: something like: ```eval "$(direnv hook bash)"``` (check ```direnv``` manual for more information).

- set up the ```.envrc``` file in the folder that should be used together with a conda environment; make sure to i) activate conda, ii) activate the right environment, iii) provide a ```DIRENV_PREFIX``` if you want your prompt to remind you that a conda env is activated by direnv using a small hack in your ```.bashrc``` (more on that later); a typical example would be:

```bash
eval "$(/home/jrmet/miniconda3/bin/conda shell.bash hook)"  # activate conda; adapt your path / command
conda activate test_env  # choose which env to activate
export DIRENV_PREFIX="test_env "  # show it on your promt "with a hack"; see under; note the space!
```

- you can trust your own ```.envrc```, so you can confidently ```direnv allow .``` it.

- there are a few subtleties getting the PS1 prompt to indicate that a conda environment is activated by ```direnv``` (basically, PS1 is a local variable, that is evaluated by your shell as a normal variable first, before being interpreted as a PS-syntaxed string). To make this work, on bash, you can for example have something in this kind inside your ```.bashrc```, after your other "PS1 magics":

```bash
# we want bash to substitute DIRENV_PREFIX for us as a normal variable,
# but not the previous content of PS1, that should be handed in to the "PS-parser" as is from before
PS1='${DIRENV_PREFIX:-}'"$PS1"
```

If you use some more advanced prompt, for example the ```git-prompt``` [https://github.com/git/git/blob/master/contrib/completion/git-prompt.sh](https://github.com/git/git/blob/master/contrib/completion/git-prompt.sh), this "PS1 hacking" may not be the correct way to display your env status - see your prompt and prompt magics user manual for more information.

Note that if you want to turn down ```direnv``` verbose "loading" and "export" messages you can add the following to your ```.bashrc```:

```bash
export DIRENV_LOG_FORMAT=""
```

The magics should now work:

```bash
~$ cd Desktop/Current/Test_Direnv/
test_env ~/Desktop/Current/Test_Direnv$ conda info

     active environment : test_env
     [...]

test_env ~/Desktop/Current/Test_Direnv$ cd ..
~/Desktop/Current$ conda info
conda: command not found

```

## Best practice: don't annoy your admin!

This is really convenient, but also opens the door for possible abuse. Make sure not to annoy your admin, and make sure that what you do complies with the policies set for using the server.

In particular, check that it is ok to create your own conda environments in the first place, and that it is ok to use the location that you consider (disk use and bandwidth can add up quickly; for example, a simple conda environment with a couple of python packages I use for plotting figures is using a total disk space of around 1.7GB! Remember that you can use the ```du -h .``` command from the base folder of a conda environment to get its disk use). Remember that having a multitude of conda environments can quickly eat up a lot of disk: make sure to clean your old un-used conda environments when relevant.

Also make sure not to run out-of-date software that can contain bugs and vulnerabilities, so as not to put your user account and the server in danger.
