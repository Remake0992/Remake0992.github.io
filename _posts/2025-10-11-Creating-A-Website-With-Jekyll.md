---
title: 2025-10-11-Creating a website with Jekyll
date: 2025-10-11 15:05:00 
categories: [homelab,selfhosted,professional-development]
tags: [servers,sites,selfhosted]
---

# Creating a Website with Jekyll

Welcome to the first post of my personal website! I have been wanting to create a portfolio-ish website for a while now. I followed Techno Tim's guide on YouTube to create the website using GitHub and GitHub pages.

This will be the first project documented in the site. Here's how it came to be!

## Starting Off
First, I installed my dependencies. This was done on a fresh Ubuntu 24.04 VM.

```shell
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev git
```

Then, I installed the Jekyll Bundler.

```shell
gem install jekyll bundler
```

## Creating the Site
Now it was time to make my website. If you weren't aware, Jekyll is a project formulated by Github. It servers as a static site generator, and has many different themes created by the community. I chose Chirpy for my theme. It's simplistic and very lightweight. I was almost inspired by James Scholz to go with an entirely html-based site but changed my mind in the end. It's all about the style points, y'know? 

After I follows the instructions at https://github.com/cotes2020/jekyll-theme-chirpy#quick-start I cloned the repository. 

```shell
git clone git@Remake0992/Remake0992.github.io.git
```

After that, I had some dependencies to install

```shell
cd Remake0992.github.io.git
bundle
```

Then I edited the config file to showcase the information I wanted. I removed links to social media, as I don't really have any besides LinkedIn. 

After that, all there was to do was to commit and push!

```shell
git add .
git commit -m "configuring my site for the first time"
git push
```

## Going forward...
I can also serve the site on my VM using

```shell
bundle exec jekyll s
```

To build in production, I'd use

```shell
JEKYLL_ENV=production bundle exec jekyll b
```

## Wrapping up
In any case, that's all for now. I guess all I really needed was one more thing to tinker with. Feel free to explore whatever content is available at the time of reading this.
