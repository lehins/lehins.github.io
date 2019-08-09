# How this website was setup

I think it is an appropriate topic for the very first blog post on my personal website. It
might come in handy for anyone who'd like to setup a personal website for free using
[github.com](https://github.com){:target="_blank"} hosting and
[Jekyll](https://jekyllrb.com){:target="_blank"} for rendering. I can even point here
people I know that might ask me how I've set it up. More importantly though, I will
probably find it useful later on myself if I ever need to replicate these steps for some
odd reason. There are likely better guides out there than this one, but I'll let you be
the judge of that.

## Choose the template

First things first, please your eyes. You probably already know what will be the content
of your static website, so you have some ideas for the style, the looks and the
layout. There are plenty of resources online with Jekyll templates. FOr instance the one you see
here is called ["Simple Texture"](http://jekyllthemes.org/themes/simple-texture/){:target="_blank"}
and I found it on [JekyllThemes.org](http://jekyllthemes.org){:target="_blank"}.
Ultimately the goal is to find a repository with a Jekyll template you'd like to use as the
base for your site. Once you find one, you need to make a copy for yourself. In my case
I made a fork of the repo with above mentioned template and used it as the beginning of
the history for my new site's repository.

## Hello World

Before we start populating the new website with some cool content we need to set it up and
make it accessible to the world first. Here are the steps to achieve it:

* Create a [github.com](https://github.com/join){:target="_blank"} account, if you don't have one already.
* Fork the template repo that you, as mentioned before.
* Go into the "Settings" of your newly forked repository and change its name. Type in
  `<your-github-username>.github.io` and hit "Rename". That is the reason why my website's
  repository is named
  [`lehins.github.io`](https://github.com/lehins/lehins.github.io){:target="_blank"},
  because the name of the repo for your free website must match you github username. Which
  means, you can only host one such free website.
* Make sure "Github Pages" are also setup in the setting for the repository. You should
  see a green box saying something like: "Your site is published at
  https://lehins.github.io/"
* Go to `<your-github-username>.github.io` and Booyah! You will see the template you chose
  for your website.

## Your own domain (Optional)

In you case you do have some money to burn and you'd like to give it a personal feeling, you
might wanna consider buying a domain name for a few bucks a year and point it to your
newly created site. Your journey will likely be different on how you do it, but in the end
you need to create a `CNAME` record that points to you
`<your-github-username>.github.io`. Go to the website where you bought you domain and add
a settings along those lines:

|  Type   |  Host  |            Value                 |  TTL  |
|:-------:|:------:|:--------------------------------:|:-----:|
|  CNAME  |  www   | <your-github-username>.github.io | X min |

In my case I have a subdomain, so I've set it to `alexey` instead of `www`:

|  Type   |  Host  |      Value       |  TTL  |
|:-------:|:------:|:----------------:|:-----:|
|  CNAME  | alexey | lehins.github.io | 5 min |


Now got the setting of your site's repository on GitHub and go to "GitHub Pages"
section. Aftrewards you need go into GitHub settings page and specify the same domain name
as the one you sued with your domain provider, for example in my case it was
`alexey.kuleshevi.ch`. Note that updating that setting will change a `CNAME` file in your
repository. But most importnatly it will make sure that when you visit your domain name in
the browser's address bar, it will take you to your new badass website.

If you like security as much as I do then you will wanna check the box for "Enforce HTTPS"
setting. This way everyone trying to visit an unencrypted version of your website, eg.
`alexey.kuleshevi.ch` will automatically get redirected to the secure version of it
`https://alexey.kuleshevi.ch` instead.

## Make it unique

Thus far we got a clone of someone else's work. What we need to do now is add some changes
to the site. There are two modes of getting things done here, a grandpa way where you
change files in the GitHub UI by clicking "Edit this file" button, or by doing it the
right way and setting up a local copy of your new website. The latter gives you a couple
of advantages:

* viewing the changes you make locally before the live website gets updated
* ability to use your favorite text editor to work on the content

At this point I will assume you are setting this stuff up on Ubuntu as I do, because
otherwise your adventure from now on will likely be be different.

Go to your home directory and create a folder named `github`:

```shell
$ cd ~
$ mkdir -p github
$ cd github
```

We need to get our new website onto the local file system, so we can make changes to it
and later update the remote version, which everyone else can see. In order to do that, we
need to get `git` installed and get local clone of our repository:

```shell
$ sudo apt-get install -y git
$ USERNAME="<your-github-username>"
$ git clone git@github.com:$USERNAME/$USERNAME.github.io.git
$ cd $USERNAME
```

Git will likely ask you for your GitHub username and password. I would recommend
[setting up an ssh key](https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent){:target="_blank"}
for working with your GitHub repositories, but that is a totally separate subject and I'll let
you figure it out on your own.

The important part is that now you can edit files present in the repository using your
favorite text editor. For me it is [`emacs`](https://www.gnu.org){:target="_blank"}, but
you can use whichever one makes you happy. First place to start, I'd say, is a
`_config.yml` file in the repo. Change the info you find the way you see fit, `commit` those
changes to the repo and then `push` them to the remote repository:

```shell
$ git commit -am "First change to my new website. Gave it a proper name and set the links"
$ git push
```

Now all you gotta do in order to see the updates to your website is give GitHub a few
minutes to react to the changes and it all will get updated automatically. Keep in mind, more
complexity you add to the site, longer it will take for GitHub to reflect the changes, but
it is never too long.

## Add new blog post

As I've been setting my website up I was keeping this document up to date. Now it looks
like it is time to add it to the repository and publish my first draft:

```shell
$ mv ~/personal-website-setup.md _posts/2019-08-09-personal-website-setup.md`
$ git add -A
$ git commit -am "Attempt to publish a draft of my first website"
$ git push
```

There are still a couple more sections I want to finish, but I'll do them with separate
commits.

## Local development

Going through the whole process of committing to the repository and pushing to the remote
server just to see any slight change you make is a bit inconvenient. We can handle this
problem by installing local development environment and then we will be able to check
changes locally before pushing them to the remote server. All we need is just a few tools
installed locally. Ruby runtime is the first one:

```shell
$ sudo apt-get install -y ruby ruby-dev
```

You can follow other tutorials and install the `bundler` and `jekyll` as sudo, but I would
advise against it and go through a few extra steps:

```shell
$ mkdir ~/.ruby
$ echo 'export GEM_HOME=~/.ruby/' >> ~/.bashrc
$ echo 'export PATH="$PATH:~/.ruby/bin"' >> ~/.bashrc
```

Logout then log back in again, although for current shell session you can just do `$ source
~/.bashrc`. After which you can installed gems without root permissions:


```shell
$ gem install bundler jekyll
```

Next you need to install all dependencies that your template and Jekyll site need:

```shell
$ bundle install
```

After which all you that separates you from seeing your website locally in your browser is
the commands that starts up a ruby development webserver:

```shell
$ bundle exec jekyll serve
Configuration file: /home/lehins/github/lehins.github.io/_config.yml
            Source: /home/lehins/github/lehins.github.io
       Destination: /home/lehins/github/lehins.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 0.676 seconds.
 Auto-regeneration: enabled for '/home/lehins/github/lehins.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
```

Go on and open you favorite browser and type in
[`http://127.0.0.1:4000`](http://127.0.0.1:4000){:target="_blank"} and you ought to see
your website, that is running locally. As instructed above, just hot `Ctrl-C` to
stop. Cool thing about it is, that whenever you change any files in the project, Jekyll
will sense it and will generate new contents as soon as you save the file.

# Conclusion

Not much to say here, hopefuly these instructions turned out to ne useful for you. Wish
you luck with sharing your knowledge with the world.
