---
title: "This website is hosted for free. Here's how."
date: 2024-12-24T17:52:13Z
draft: false
images: 
- this-website-is-hosted-for-free.gif
tags: ['github', 'devops']
summary: "I host this website for exactly $0. The domain name and email cost me money, yes, but the hosting? Zero. Zip. Nothing. Such is the magic of Hugo and Github Pages."
---

I host this website for exactly $0. The domain name and email cost me money, yes, but the hosting? Zero. Zip. Nothing. 

![Free web hosting???](/this-website-is-hosted-for-free.gif 'Free web hosting???')

# How is this possible?

The short answer is compiled html and css and github pages.

Let's get into it.

# Why not WordPress if most of your career has been in WordPress development?

I come from a background of WordPress development. That has been most of my decade plus career. 

WordPress is pretty chunky. It uses PHP, it requires a database, you need to know what you're doing in order to minify your js and css - the list goes on.

It also uses either the new block editor - which appeals to an audience more familiar with site generators - or the TinyMCE editor, which feels like a relic from 2010.

And lastly, hosting WordPress is turning into a rather impressive endeavor. You need PHP modules, auto-updates for the CMS, the database, and the PHP language, etc. That means that your average WordPress host wants $$$. 

That all said, I write a _lot_ of documentation in [markdown syntax](https://www.markdownguide.org/basic-syntax/). I'm also a big fan of saving money. 

Was it possible to simply use markdown files and compiled html and css to generate an entire site? Yes! And the answer came in the form of [Hugo](https://gohugo.io/). 

# What is Hugo?

From their own site:

> Hugo is a static site generator written in Go, optimized for speed and designed for flexibility. With its advanced templating system and fast asset pipelines, Hugo renders a complete site in seconds, often less.
>
> Due to its flexible framework, multilingual support, and powerful taxonomy system, Hugo is widely used to create:
>
> -Corporate, government, nonprofit, education, news, event, and project sites
> - Documentation sites
> - Image portfolios
> - Landing pages
> - Business, professional, and personal blogs
> - Resumes and CVs
- https://gohugo.io/about/introduction/

I have taken a simple course on Golang. I don't pretend to be an expert, and the best thing is, I don't need to be. 

This blog is using a free theme for which I have a few overrides and extensions set. I write posts as markdown files, test it locally, and then push the code to my remote repo on GitHub.

You can easily setup Hugo to [minify your code](https://gohugo.io/getting-started/configuration/) so your website is even faster. It's literally just a few lines in a `config.yaml` file. No fussing with setting up extra tools like webpack or sass - it comes built in with Hugo. 

Hugo also has a live reload ability during development. I have Hugo setup in a `docker-compose.yaml` setup that is served on `http://0.0.0.0:1313`. Anytime I write a new post, I can count on `hugo server` not only watching for new changes and re-compiling the `public` dir on change, but if I have the page already up in my browser, it'll automatically reload with no prompting by me. 

Worried about [RSS](https://gohugo.io/templates/rss/#configuration) feeds? Hugo has that.

A [search](https://gohugo.io/tools/search/) capability? Hugo has that.

The ability to use [shortcodes](https://gohugo.io/content-management/shortcodes/) in articles? Hugo has that too.

The exact code behind this blog is available here: https://github.com/kimdcottrell/techlab 

# Free hosting on GitHub Pages

GitHub pages - at least at the time of this writing - let's you host one repo on your account as a site for _free_.

**It only allows for static html, css, and javascript though. No databases.** 

Since Hugo compiles down to just html, css, and javascript in a directory called `public`, you can use GitHub Pages to serve everything in that `public` dir.

Hugo provides instructions on how to setup things on GitHub's end here: https://gohugo.io/hosting-and-deployment/hosting-on-github/ 

If you want to use a custom domain name instead of github's generated url of `http(s)://<username|organization>.github.io`, you'll have to follow some additional steps available here: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site 

# How does deployment look for this blog

When I push my code to my remote repo, github sees that I have an action defined in `.github/workflows`. 

It runs the action available here: https://github.com/kimdcottrell/techlab/blob/main/.github/workflows/hugo.yaml

Those actions show up in the `Actions` tab here: https://github.com/kimdcottrell/techlab/actions 

Those actions will either error out and the deployment will be cancelled, or they succeed and the deployment rolls out to github pages. 

# Closing thoughts

In total, this website, domain, and email runs me maybe $20 a year total. 

I save significant time not having to fuss over WordPress's drama, updates, and monetary costs. 

If you're at all familiar with markdown syntax, I highly recommend looking into Hugo for your blog, even if you don't have a good understanding of Golang. 


