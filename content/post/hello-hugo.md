+++
date = "2017-03-14T19:45:58+01:00"
draft = false
title = "Hello hugo on github.io"
featuredImage = "hugo-logo.png"
tags        = [ "Hugo", "Hello world"]
+++

This blog is built using hugo. So my first blogpost is dedicated to how I set it up on github. 

# installation
install the bleeding edge hugo version by running 

    go get -v github.com/spf13/hugo
    
verify the installation by typing

    hugo -h
   
this should display the help screen for hugo


# create hello world 

## create blog hosted on github.io
On github, create a repository _"\<yourusername\>.github.io"_, e.g. _"nilsmagnus.github.io"_ .

Clone the repo and cd into the directory.

From the root of your repository, run the command 

    hugo new site blog

This will create a directory , _blog_ , in your repo with the needed files

## add a theme and config

Since hugo does not have a default theme, you need to get one. This can be changed at a later point. 

    cd <repo>/blog/themes && git clone https://github.com/dim0627/hugo_theme_robust.git 

To avoid submodule conflicts with github, delete the _.git_ directory in _\<repo\>/blog/themes/hugo\_robust\_theme_. 

To specify that you want to use the theme, edit _\<repo\>/blog/config.toml_, add these lines

    baseUrl = "https://<yourusername>.github.io"
    title = "Ubeercool blog name"
    theme = "hugo_theme_robust"
    publishDir = "../"

This will also set the name of your blog and publish the blog to the parent directory (_"\<repo\>/"_).

## add a first blogpost

Add a first blogpost by typing (from _\<repo\>/blog_)

    hugo new post/hello-world.md
    
Edit this file by adding the following lines:

    +++
    date = "2017-03-14T16:11:58+05:30"
    draft = false
    title = "Hello hugo world"
    +++
    
    # Hello world

    You look _beautiful_ in markdown.

Now run hugo without parameters, 

    hugo

This will generate the entire blog. Now, add all your files(generated and source) to git and push the changes. 

**done**
 

    
