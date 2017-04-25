---
layout: post
title: Using Jekyll On Windows
---

Like most [GitHub Pages](https://pages.github.com/) websites, [this one](https://ptupitsyn.github.io/) uses [Jekyll](https://jekyllrb.com/) to generate HTML content from Markdown files.

Latest Windows update improves [WSL](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) a lot, and Jekyll now works seamlessly.

1. Install WSL: [msdn.microsoft.com/en-us/commandline/wsl/install_guide](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide),
make sure to update packages: `sudo apt-get update`

2. Install Ruby and build tools: `sudo apt-get install build-essential ruby ruby-dev`

3. Install Jekyll: `gem install jekyll jekyll-sitemap jekyll-feed` (DO NOT install jekyll package with apt-get)

4. Switch to your site root and run `jekyll serve`

That's it!