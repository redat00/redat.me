---
title: "Install Hugo With Ansible"
date: 2022-10-08T13:58:47+02:00
description: "Hugo is a an open-source framework used to build website. It is a small tool that you can install directly on your machine, or as a tool on your server. The idea is that it will create a static version of a given website, based on content you will feed him."
summary: "Hugo is a an open-source framework used to build website. It is a small tool that you can install directly on your machine, or as a tool on your server. The idea is that it will create a static version of a given website, based on content you will feed him."
draft: false
categories: ["ansible", "hugo"]
showSummary: true
---

[Hugo](https://github.com/gohugoio/hugo) is a an open-source framework used to build website. It is a small tool that you can install directly on your machine, or as a tool on your server. The idea is that it will create a static version of a given website, based on content you will feed him. Basically Hugo is just a small CLI tool, and nothing more. You will also have an option to serve content and run a small webserver, but it is more for debugging and previewing content that anything more. 

<img src="featured.png"/>

Knowing all of this, installing Hugo is pretty easy, so easy that it could only be a single task in Ansible. But I will go a deep further by showing you how to create a website directly in Ansible, and configure it, and even how to add themes.

## Creating a role

Since I knew I might have variables, templates, handlers and stuff like this for my setup to work, I created a role in my `roles/` folder. While many recommend it, I didn't use the `ansible-galaxy init` method, I'd rather just create the folder by hand. This is the command I used to create the `hugo` role, you're free to use it or no, but note that through this post I will be using this structure.

```bash
cd roles
mkdir -p hugo/tasks
mkdir -p hugo/templates
mkdir -p hugo/defaults
```

```bash
hugo
├── defaults
├── tasks
└── templates

3 directories
```

## Hugo Installation

Hugo can be installed various way, depending on what OS you're running. Here since we are talking about Ansible, I'm assuming that you want to install it on a distant machine, a server. But to be fair I could be wrong, since some people, like Jeff Geerling, apparently use Ansible to [configure their own laptop!](https://www.youtube.com/watch?v=1VhPVu5EK5o)

In my case, I was trying to install it on a Debian/Ubuntu environment. Scrolling the official Hugo documentation I learned that there was actually a package maintained for this specific environment, and that the only thing to do was to install it like I would normally do with `apt-get install hugo`. 

This is what I did in a first time, before seeing, when I had trouble to install a theme, that this package is way behind in term of release version. The version that was available in the official Debian repository was `0.80.0`, while the latest release of Hugo is `0.104.3`.

Knowing this I had to change plan, and install Hugo by directly downloading it on GitHub. 

The first thing I did was to define inside of the `defaults/` folder, the Hugo version I wanted to install. I did so by just creating a `main.yml` and setting the `HUGO_VERSION` variable inside it. 

```yaml
---

HUGO_VERSION: 0.104.3
```

The first task I wrote was a simple one that would get the `*.deb` package from the internet, and directly install it. Hugo developers actually make it really easy to install it from GitHub, since they're proposing `deb` packages directly for download.

```yaml
---

- name: Install Hugo deb package directly from GitHub
  ansible.builtin.apt: 
    deb: "https://github.com/gohugoio/hugo/releases/download/v{{ HUGO_VERSION }}/hugo_{{ HUGO_VERSION }}_linux-amd64.deb"
```

If you stopped here, you'll be fine, and you will have Hugo installed on your server. But as I said in the intro, I wanted to go a little deeper, and create a website with Hugo directly with Ansible. 

## Create a website using Ansible

To use Hugo, you should read the [official documentation](https://gohugo.io/documentation/) which is really well written. While reading the documentation is something you should do, as always, using Hugo is not that hard, and you only need to know a few commands. 

All of the following command, except `hugo new site` should be run inside of an already created Hugo site, otherwise they will simply result in an error.

| Command | Description |
| ------- | ------------- |
| hugo new site [path] | This command is the one you will need to only run once. It is simply, as it says, to create a new website. You can give him an absolute path, but if you only give him a site name for example, it will just create a folder with the given name where you're standing. |
| hugo server | This command will generate the static website and a small webserver locally that you can access at [http://localhost:1313/](http://localhost:1313/). |
| hugo -D | This command will simply generate the static content, and put it, by default, inside of the `public/` folder. |

I'm just giving you the one you'll see in the next lines, a lot of other commands exists, you can run `hugo help` to learn about them.

Since Hugo creates a folder when it create a website, we will be using this as an advantage. When you use some Ansible functions, you have an arguments that is called `creates`. It is very useful because sometimes you don't want command to run if something is already done. 

In my case I simply wrote a small task that run `hugo new site` if given folder does not exist, otherwise it will be skipped.

```yaml
- name: Create website if folder does not exists
  command:
    cmd: hugo new site /var/www/example.com
    creates: /var/www/example.com
```

As a result, you should have a new folder at the given path containing some folders, and files.

```bash
example.com
├── archetypes
│   └── default.md
├── config.toml
├── content
├── data
├── layouts
├── public
├── static
└── themes

7 directories, 2 files
```

In the beginning steps I made you create a `templates/` folder inside of the hugo role, we will be using it know to manage the `config.toml` that Hugo created. By default this file contains little to nothing in terms of configuration, it's simply 3 lines. 

```toml
baseURL = 'http://example.org/'
languageCode = 'en-us'
title = 'My New Hugo Site'
```

I want to make this file a template inside of my Ansible role, so I decided to create `config.toml.j2` (`.j2` being the extension we give to let others know this is a Jinja2 template file). 

```toml
baseURL = "{{ HUGO_BASE_URL }}"
languageCode = "{{ HUGO_LANGUAGE_CODE }}"
title = "{{ HUGO_SITE_TITLE }}"
```

Then, in the supposedly already existing `templates/main.yml` files, I will add some variables.

```yaml
---

HUGO_VERSION: "0.104.3"
HUGO_BASE_URL: "https://example.com"
HUGO_LANGUAGE_CODE: "en-us"
HUGO_SITE_TITLE: "Example Site"
```

And finally, in the `tasks/main.yml` I will be adding a new task that will take my newly created template, and put it in the correct folder, in place of the previous `config.toml` file.

```yaml
- name: Copy config.toml template file
  ansible.builtin.template:
    src: "config.toml.j2"
    dest: "/var/www/example.com/config.toml"
```

## Adding a theme

With Hugo, as for everything, it is quite easy to install a theme. The only real thing you have to do, is actually to download did inside of the `themes/` folder, and then set it to be used by Hugo. 

To do so I personnaly used the `ansible.builtin.git` functions, since the theme I wanted, [Blowfish](https://github.com/nunocoracao/blowfish), was available on GitHub. The [official Blowfish documentation](https://nunocoracao.github.io/blowfish/docs/installation/#install-using-hugo) suggests other way of doing it, but I find my solution easier to do, especially using Ansible. The task will clone the theme repository inside of the `themes/` folder, and not doing anything (not even an update) if the folder already exists.

```yaml
- name: Install Blowfish themes
  ansible.builtin.git:
    repo: "https://github.com/nunocoracao/blowfish.git"
    dest: "/var/www/example.com/themes/blowfish"
    update: no
```

In order to tell Hugo to use this new theme, you simply have to add the following line to your `config.toml` file. 

```toml
theme = 'blowfish'
```

If you followed all the steps of this post, you should know have a set-up Hugo website ready to be used, with a nice looking theme, only using Ansible. You could go deeper in the Ansible way, and use it to deploy your website content since they're only Markdown file, but personnaly I only use Ansible to set-up the application, and the content if it's stored on Git or an S3, I don't want to mix applications set-up and content. 

I hope you found this post useful! Until the next time, take care!

