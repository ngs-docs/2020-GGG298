# Advanced Intro UNIX and integration

## Learning Objectives
----------------------
* 


## Part 1: Sourmash project revisited

[Let's pick up our sourmash project where we ended last week.](https://hackmd.io/XgI03HNBRtS6kyKcycKFLA?view#4-Add-your-files-to-git-and-push-to-github)




## Part 2: File Manipulation
-------------------------------

### Learning Goals
* combine commands to carry out a sequence of steps with `|`
* redirect output of commands to files with `>`
* increase exposure to regular expressions
* commands `for`, `basename`, `echo`
------------------------------------------

### Renaming a bunch of files

Let's grab some data:
```
wget https://s3-us-west-1.amazonaws.com/dib-training.ucdavis.edu/shell-data.zip
unzip shell-data.zip
cd data/MiSeq
```

For our first task, let's pretend that we want to rename all of the fastq files to be `.fq` files instead (this is a surprisingly useful specific skill, even if you can't immediately think of why you would want to do that!).  Here, we get to use two of my favorite commands - 'for' and 'basename'.

`for` lets you do something to every file in a list.  To see it in action:

```
for i in *.fastq
do
   echo $i
done
```

This is running the command `echo` for every value of the variable 'i', which is set (one by one) to all the values in the expression `*.fastq`.

If we want to get rid of the extension '.fastq', we can use the `basename` command:

```
for i in *.fastq
do
   basename $i .fastq
done
```

Now, this doesn't actually rename the files - it just prints out the name, with the suffix '.fastq' removed.  To rename the files, we need to capture the new name in a variable::

```
for i in *.fastq
do
   newname=$(basename $i .fastq).fq
   echo $newname
done
```

What `$( ... )` does is run the command in the middle, and then replace the `$( )` with the output of running the command.

Now we have the old name ($i) and the new name ($newname) and we're ready to write the rename command -- ::

```
for i in *.fastq
do
   newname=$(basename $i .fastq).fq
   echo mv $i $newname
done
```

***Question:*** why did I use `echo` here?

Now that we're pretty sure it all looks good, let's run it for realz:

```
for i in *.fastq
do
   newname=$(basename $i .fastq).fq
   mv $i $newname
done
```

and voila, we have renamed all the files!

_Side note:_ you may see backquotes used instead of `$(...)`. It does the same thing but is trickier to get right, so we teach `$(...)` instead of ``...``.

----

Now let's also get rid of the annoying '\_001' that's at the end of the all files.  `basename` is all fine and good with the end of files, but what do we do about things in the middle? Now we get to use another one of my favorite commands -- `cut`.

What `cut` does is slice and dice strings.  So, for example, :

```
echo hello, world | cut -c5-
```

will print out `o, world`.

But this is kind of a strange construction! What's going on?

Well, `cut` expects to take a bunch of lines of input from a file. By default it is happy to take them in from stdin ("standard input"), so you can specify '-' and give it some input via a pipe, which is what we're doing with echo:

We're taking the output of 'echo hello, world' and sending it to the input of cut with the `|` command ('pipe').

You may have already seen this with head or tail, but many UNIX commands take stdin and stdout.

Let's construct the `cut` command we want to use.  If we look at the names of the files, and we want to remove '001' only, we can see that each filename has a bunch of fields separated by '\_'.  So we can ask 'cut' to pay attention to the first four fields, and omit the fifth, around the separator (or delimiter) '\_':

```
echo F3D141_S207_L001_R1_001.fq | cut -d_ -f1-4
```

That looks about right -- let's put it into a for loop:

```
for i in *.fq
do
   echo $i | cut -d_ -f1-4
done
```

Looking good - now assign it to a variable and append an ending:

```
for i in *.fq
do
   newname=$(echo $i | cut -d_ -f1-4).fq
   echo $newname
done
```
  
and now construct the `mv` command:

```
for i in *.fq
do
   newname=$(echo $i | cut -d_ -f1-4).fq
   echo mv $i $newname
done
```

and if that looks right, run it:

```
for i in *.fq
do
   newname=$(echo $i | cut -d_ -f1-4).fq
   mv $i $newname
done
```

Ta-da! You've renamed all your files.

----

### Subsetting

Let's do something quite useful - subset a bunch of FASTQ files.

If you look at one of the FASTQ files with head, 

```
head F3D0_S188_L001_R1.fq
```

you'll see that it's full of FASTQ sequencing records.  Often I want to run a bioinformatices pipeline on some small set of records first, before running it on the full set, just to make sure all the syntax for all the commands works. So I'd like to subset all of these files without modifying the originals.

First, let's make sure the originals are read-only

```
chmod u-w *.fq
```

Now, let's make a 'subset' directory

```
mkdir subset
```

Now, to subset each file, we want to run a 'head' with an argument that is the total number of lines we want to take.  In this case, it should be a multiple of 4, because FASTQ records have 4 lines each. Let's take the first 100 records of each file by using `head -400`.

The for loop will now look something like:

```
for i in *.fq
do
   echo "head -400 $i > subset/$i"
done
```

If that command looks right, run it for realz:

```
for i in *.fq
do
   head -400 $i > subset/$i
done
```

and voila, you have your subsets!

(This is incredibly useful. You have no idea :)

----

**CHALLENGE:** Can you rename all of your files in subset/ to have 'subset.fq' at the end?

(Work in small groups; start from working code; there are several ways to do it; all that matters is getting there!)

A little backtracking...
------------------------

Variables:

You can use either $varname or ${varname}.  The latter is useful when you want to construct a new filename, e.g.

```
   MY${varname}SUBSET
```

would expand ${varname} and then put MY .. SUBSET on either end, while 

```
MY$varnameSUBSET
```

would try to put MY in front of $varnameSUBSET which won't work.

(Unknown/uncreated variables are empty.)

NOTE: `${varname}` is quite different from `$(expression)`! The former is replaced by the value assigned to `varname`; the latter is replaced by the result of running `expression`. So, both _replace_ but they do different things. Think of `$` here as meaning, "replace me with something".

---

We used "$varname" above - what happens if we use single quotes -
e.g. '$varname'?

(Variables are interpreted inside of "", and not inside of ''.)

----

### Pipes and redirection

To redirect stdin and stdout, you can use:

```
> - send stdout to a file
< - take stdin from a file
| - take stdout from first command and make it stdin for second command
>> - appends stdout to a previously-existing file
```

stderr (errors) can be redirected:

```
2> - send stderr to a file
```

and you can also say::

```
>& - to send all output to a file
```

### Editing on the command line:

Most prompts support 'readline'-style editing.  This uses emacs control
keys.

Type something out; then type CTRL-a.  Now type CTRL-e.  Beginning and end!

Up arrows to recall previous command, left/right arrows, etc.

CTRL-r will search backwards for recent commands, too! If you find the exact
command you are looking for, you can hit ENTER to run it again. If you
want to edit it, use CTRL-a or CTRL-e to move to editing mode.

----

CHALLENGE: Another useful command along with 'basename' is 'dirname'. Any idea what
it does?

-----

Working with collections of files; conditionals
-----------------------------------------------

Let's go back to the 'data' directory and play around with loops some more. 

```
  cd ..
```

`if` acts on things conditionally:

```
for i in *
do
   if [ -f $i ]; then
      echo $i is a file
   elif [ -d $i ]; then
      echo $i is a directory
   fi
done
```

but what the heck is this `[ ]` notation?  That's actually running the 'test' command; try `help test | less` to see the docs.  This is a weird syntax that lets you do all sorts of useful things with files -- I usually use it to get rid of empty files.

```
touch emptyfile.txt
```

to create an empty file, and then::

```
for i in *
do
   if [ \! -s $i ]; then
      echo rm $i
   fi
done
```

...and as you can see here, I'm using '!' to say 'not'. (Why do I need to put a backslash in front of it, though??)

Executing things conditionally based on exit status
---------------------------------------------------

Let's create two scripts (you can use 'nano' here if you want) -- in 'success.sh', put:

```
  #! /bin/bash
  echo mesucceed
  exit 0
```

and in 'fail.sh', put:

```
  #! /bin/bash
  echo mefail
  exit 1
```

You can do this with 'heredocs' -- ::

```
  cat > success.sh <<EOF
  #! /bin/bash
  echo mesucceed
  exit 0
  EOF
  cat > fail.sh <<EOF
  #! /bin/bash
  echo mefail
  exit 1
  EOF
 ```

Now make them executable -- 

```
chmod +x success.sh fail.sh
```

(Somewhat counterintuitively, an exit status of 0 means "success" in UNIX land. Any non-zero exit status is an error.)

You can now use this to chain commands with `&&` and `||` -- :

```
./success.sh && echo this succeeded || echo this failed
./fail.sh && echo this succeeded || echo this failed
```

You can do this with R and python scripts too -- in R, you set the exit status of a script with `quit(status=0, save='no')` and in Python with `sys.exit(0)`.  Any failure of the script due to an exception will automatically set the exit status to non-zero.

The exit status of the previous command can be examined with `$?` -- :

```
./success.sh
if [ $? -eq 0 ]; then echo succ; fi
```

```
./success.sh
if [ $? -ne 0 ]; then echo fail; fi
```

Writing shell scripts
---------------------

Always put `set -e` at the top.

Sometimes put `set -x` at the top.

You can take in command line parameters with '$1', '$2', etc. '$\*' gives you all of them at once.

Other things to mention
-----------------------

Scripts exit in a subshell and can't modify your environment variables. If you want to modify your environment, you need to use '.' or 'source'.

Subshells are ways to group commands with ( ... ).

You can use \ to do line continuation in scripts (in R and Python, too!)

History tricks:

```
  !! - run previous command
  !-1 - run command-before-previous command (!-2 etc.)
  !$ - replace with the last word on the previous line
  !n - run the nth command in your 'history'
```


The general approach I use
--------------------------

* break the task down into multiple commands
* put commands things in shell scripts, run in serial
* use intermediate i/o files to figure out what's going on!
* use echo to debug!

(But we'll also teach you another way to do automation with snakemake.)



## Part 3: Integration
----------------------

@@ctb

Topics will include:
* how does a computer work
* practical next steps for everyone
    * workshops, online sites, books, etc
