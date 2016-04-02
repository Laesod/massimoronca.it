---
title: Create an animated GIF of your console sessions
date: "2014-11-05"
description: "More often than not, our job involves opening up a console, typing some command and waiting for the output.   
When I write articles, sometimes I feel the need to show how the commands behave interactively, not only the sequence of commands you have to type.  
Fortunately, the solution is pretty easy."
image: http://s2.quickmeme.com/img/32/325fc351053e41d230961a71308d37937e68192130d11a82308ae619571ef942.jpg
tags:
- animated gif bash
- console session animated gif
- ttyrec
- ttyrplay
- ttygif
- gifsicle
---



{{% figure src="http://s2.quickmeme.com/img/32/325fc351053e41d230961a71308d37937e68192130d11a82308ae619571ef942.jpg" title="animate all the things" %}}

More often than not, our job involves opening up a console, typing some command and waiting for the output.   
When I write articles, sometimes I feel the need to show how the commands behave interactively, not only the sequence of commands you have to type.  
It's easier to understand by looking at an animation, than reading  
"when you hit TAB \<this happen\>".  

For example, can you explain how the emmet plugin for VIM works, better than this, using only words?   

{{% figure src="https://qiita-image-store.s3.amazonaws.com/0/38647/86b91c27-f894-c969-89b0-5846408ad1db.gif" title="emmet plugin for VIM" %}}

Fortunately, the solution is pretty easy.   
You need a few open source tools, if you're on a Mac, like me, you should already have installed [Homebrew](http://brew.sh/), and Imagemagick (`brew install imagemagick`).  
 
To record your sessions, you need `ttyrec` (`brew install ttyrec` on Mac).  
The usage is very simlpe  

```bash
usage: ttyrec [-u] [-e command] [-a] [file]

OPTIONS
       -a     Append the output to file or ttyrecord, rather than overwriting it.

       -u     With this option, ttyrec automatically calls uudecode(1) and  saves  its  output  when  uuencoded  data
              appear on the session.  It allow you to transfer files from remote host.  You can call ttyrec with this
              option, login to the remote host and invoke uuencode(1) on it for the file you want to transfer.

       -e command
              Invoke command when ttyrec starts.
```

`file` is the name of the file that will be used to record the session. If no file name is given, `ttyrecord` will be used.   

A new session is started as soon as you launch `ttyrec` and is automatically saved when you close the session with `CTRL+D` or `exit`.   

To replay an already saved session, use 

```bash
ttyplay [-s SPEED] [-n] [-p] file

OPTIONS
       -s SPEED
              multiple the playing speed by SPEED (default is 1).

       -n     no wait mode.  Ignore the timing information in file.

       -p     peek another person's tty session.
```



Let's see how it looks

{{% figure src="http://i.imgur.com/q7NHxN0.gif" title="ttyrec session" %}}

Now that we have a recorded sessions, we need to convert it to an animated GIF.   
We'll use [`ttygif`](https://github.com/icholy/ttygif) for the task.  
There's no installer for it, you must compile it from the sources

```bash
$ git clone https://github.com/icholy/ttygif.git
$ cd ttygif
$ make
```

Once make is done, you will find some executable files in the folder

```bash
-rwxr-xr-x   1 maks  staff    881 Nov  5 16:08 concat.sh
-rwxr-xr-x   1 maks  staff    829 Nov  5 16:08 concat_osx.sh
-rwxr-xr-x   1 maks  staff  14836 Nov  5 16:08 ttygif
```

In my setup I've linked `ttygif` to `/usr/local/bin`  and `concat.sh` (`concat_osx.sh` in case you're on a Mac) to `/usr/local/bin/ttyconcat` to avoid name clash.  
Creating the gif is a two steps process: first you launch `ttygif <recfile>` to genearet a sequence of PNGs, then you launch the `ttyconcat` script we linked before and it automatically creates the animated GIF for you.  
Optionally you can pass an output filename to `ttyconcat`, if omitted the image will be saved as `output.gif`.  
  
If you are a prefectionist, there's an optional final step, install [`gifsicle`](http://www.lcdf.org/gifsicle/) (`brew install gifsicle`) and give your animation the final touches.   
I usually add a fixed delay between frames, make it loop forever and optimize the size with this command line

```bash
gifsicle --delay=10 --loop=0 -O3 < in.gif > out.gif
```

More otpions can be found on the [`gifsicle` man page](http://www.lcdf.org/gifsicle/man.html).

