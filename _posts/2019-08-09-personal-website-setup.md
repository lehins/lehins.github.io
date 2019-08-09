# How this website was setup

I think it is an appropriate topic for the very first blog post on my personal website. It
might come in handy for anyone who'd like to setup a personal website for free using
[github.com](https://github.com) hosting and [Jekyll](https://jekyllrb.com) for
rendering. I can even point here people I know that might ask me how I've set it up. More
importantly though I might find it useful later on myself if I ever need to replicate
these steps for some odd reason. There are probably better guides out there, here is the
[one](https://help.github.com/en/articles/setting-up-your-github-pages-site-locally-with-jekyll)
that I used for example, so feel free to leave this page whenever you feel like it.

## Choose the template

First things first, please your eyes. You probably already know what will be the content
of your static website, so you have some ideas for the style, the looks and the
layout. There are plenty of resources online with Jekyll templates, but the one you see
here is called ["Simple Texture"](http://jekyllthemes.org/themes/simple-texture/) and I
found it on [JekyllThemes.org](http://jekyllthemes.org). Ultimately your goal is to find a
repository with a Jekyll template you'd like to use as a base for your site. When you do
find one, you need to make a copy for yourself. In my case I made a fork of the repo with
above mentioned template and used it as the beginning of the history for my new site.

## Hello World

Before we start populating the new website with some cool content we need to set it up and
make it accessible to the world first. Here are the steps to achieve it:

* Create a [github.com](https://github.com/join) account, if you don't have one already.
* Fork the template repo that you, as mentioned before.
* Go into the "Setting" of your newly forked repository, change its name to:
  `<your-github-username>.github.io` and hit "Rename". That is why my website's repository
  is named [`lehins.github.io`](https://github.com/lehins/lehins.github.io)
* Make sure "Github Pages" are also setup in the setting for the repository. You should
  see a green box saying something like: "Your site is published at
  https://lehins.github.io/"
* Go to `<your-github-username>.github.io` and Booyah! You will see the template you chose
  for your website.

## Your own domain (Optional)

In you case you do have some money to burn and you'd like to give it personal feeling, you
might wanna consider buying a domain name for a few bucks a year and point it to your
newly created site. Your journey will likely be different on how you do it, but in the end
you need to create a `CNAME` record that points to you
`<your-github-username>.github.io`. Go to the website where you bought you domain and add
a settings along those lines:

|: Type  :|: Host :|:           Value                :|: TTL :|
|  CNAME  |  www   | <your-github-username>.github.io | X min |

For me it was a subdomain, so I've set it to `alexey` instead of `www`:

|: Type  :|: Host :|:     Value      :|: TTL :|
|  CNAME  | alexey | lehins.github.io | 5 min |


Now got the setting of your site's repository on GitHub and go to "GitHub Pages"
section. Now you need to specify the same domain name as the one from your domain
provider. In my case it is `alexey.kuleshevi.ch`. Note that updating that setting will
change a `CNAME` file in your repository. But most importnatly it will make sure that when
you visit your domain name in th browser's address bar, it will take you to your new
badass website.

If you are like security just as much as I do then I also recommend checking the box for
"Enforce HTTPS". This way everyone trying to visit an unencrypted version of the website
at `alexey.kuleshevi.ch` will automatically get redirected to the secure version at
`https://alexey.kuleshevi.ch` instead.

## Make it unique

Thus far we got a clone of someone elses work. What we need to do now is add some chages
to the site. There two modes of ding things, a grandpa way where you change files in the
GitHub UI, by clicking "Edit this file" button, or you can set up a local copy of your new
website. The latter gives you a few advantage:

* viewing the changes you make locally before the live website is updated
* ability to use your favorite text editor to work on the content

From now on, I assume you are also working on Ubuntu as I do, if not your adventure might
be different.

Go to your home directory and create a folder named `github`:

```shell
$ cd ~
$ mkdir -p github
$ cd github
```

We need to get our new website onto the local file system, so we can make changes to it
and later update the remote version, which everyone else can see. In order to do that, we
need to get `git` installed and a clone of our repository locally:

```shell
$ sudo apt-get install -y git
$ USERNAME="<your-github-username>"
$ git clone git@github.com:$USERNAME/$USERNAME.github.io.git
$ cd $USERNAME
```

Git will likely ask you for your GitHub username and password. I would recommend
[setting up an ssh key](https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
for working with your github repositories, but that is a totally separate subject and I'll let
you figure it out on your own.

The important part is that now you can edit files present in the repository using your
favorite text editor, in my case it is [`emacs`](https://www.gnu.org), but use whatever
makes you happy. First place to start I'd say is a `_config.yml` file in the repo. Change
the info you see fit, `commit` the change to the repo and `push` it to the remote repository:

```shell
$ git commit -am "First change to my new website. Gave it a proper name and set the links"
$ git push
```

Now all you gotta do in order to see the updates on your website is give GitHub a few
minutes to react to changes and all will be updated automatically. Keep in mind, more
complexity you add to th site, longer it will take for GitHub to reflect the changes, but
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

There are still a couple more sections I want to finish, but I'll do it with separate
commits.

## Local development

Going throught the whole process of committig to the repository and pushing to the remote
server just to see any slight change you make can be a bit inconvenient. We can handle this
problem by installing local development environment and then we can check changes locally
before pushing them to the remote. All we need is just a few tools installed locally:

```shell
$ sudo apt-get install -y ruby ruby-dev
```

You can follow other tutorials and install you `bundler` and `jekyll` as sudo, but I would
advise against it and go through a few extra steps:

```shell
$ mkdir ~/.ruby
$ echo 'export GEM_HOME=~/.ruby/' >> ~/.bashrc
$ echo 'export PATH="$PATH:~/.ruby/bin"' >> ~/.bashrc
```

Logout then log back in again. Or for current shell session youc an just do `$ source
~/.bashrc`. After which you can installed gems without root permissions:


```shell
$ gem install bundler jekyll
```

Next you need to install all dependencies that your template and Jekyll site need:

```shell
$ bundle install
```

After which all you that separates you from seeing your website locally in your browser is
the comands that starts up a ruby development webserver:

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
[`http://127.0.0.1:4000`](http://127.0.0.1:4000) and you ought to see your website, that
is running locally. As instructed above, just hot `Ctrl-C` to stop. Cool thing about it is that whenever you change any files in the project, Jekyll will sense it and will generate new contents as soon as you save the file.
