---
title: Github Pages with Jekyll themes and Giscus
description: How to create your personal blog using Github Pages with "comment" feature
date: 2025-08-19 01:10:00 +0700
categories: [Web]
tags: [blog, jekyll, giscus, github]
images: ["app.png"]
featuredImage: "app.png"

lightgallery: true

toc:
  auto: false
---

<!--more-->

When starting a personal blog, there are three tools you should know: GitHub Pages, Jekyll, and Giscus.
- **Github Pages**: A free service from GitHub that allows you to deploy static websites directly from a repository. Just push your code to GitHub, and your blog will automatically go live on the Internet without needing your own server.

- **Jekyll**: A static site generator integrated with GitHub Pages. Jekyll makes it easy to create blogs from Markdown files, organize content using templates, and apply ready-to-use themes.

- **Giscus**: A modern comment system based on GitHub Discussions. Instead of using external services like Disqus, you can leverage GitHub to manage comments, keeping it lightweight and developer-friendly.


## Requirements
- A [**Github account**](https://github.com/)  
- Basic knowledge of [**Markdown**](https://markdownlivepreview.com/)  


## Creating a Site Repository
Here, I use the [**Chirpy theme**](https://chirpy.cotes.page/), a popular theme for GitHub Pages optimized for personal or technical blogging.

**Steps**:  
- Log in to [**Github**](https://github.com/) and go to the [**starter**](https://github.com/cotes2020/chirpy-starter).  
- Instead of forking, click <kbd>Use this template</kbd> and select <kbd>Create a new repository</kbd> to automatically create a Site Repository.  
- Name the new repository `<username>.github.io`, replacing `<username>` with your GitHub username.  


## Setting up the Environment
There are two main reasons to set up a local development environment for your blog:  
- After pushing code to the repository, GitHub Actions takes time to run before building and rendering the site on GitHub Pages.  
- By developing directly on your local machine, you can see changes instantly—faster and more convenient.  

### Using Dev Containers (Recommended for Windows)
**Dev Containers** provide an isolated environment via Docker, preventing conflicts with your system and ensuring all dependencies are managed inside the container.  

**Steps**:  
1. Install Docker:  
   - On Windows/macOS: install [Docker Desktop](https://www.docker.com/products/docker-desktop/).  
   - On Linux: install [Docker Engine](https://docs.docker.com/engine/install/).  
2. Install [VS Code](https://code.visualstudio.com/) and the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers).  
3. Clone the repository:  
   - If using Docker Desktop: open VS Code and [clone the repo in a container volume](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-a-git-repository-or-github-pr-in-an-isolated-container-volume).  
   - If using Docker Engine: clone the repo locally, then [open it in a container](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-an-existing-folder-in-a-container) in VS Code.  
4. Wait for the Dev Containers setup process to complete.  

### Setting up Natively (Recommended for Unix-like OS)
For Unix-like systems (Linux, macOS), you can set up the environment directly (natively) for better performance. Dev Containers are still available as an alternative.  

**Steps**:  
1. Follow the [Jekyll installation guide](https://jekyllrb.com/docs/installation/) and ensure [Git](https://git-scm.com/) is installed.  
2. Clone the repository locally.  
3. If you forked the theme, install [Node.js](https://nodejs.org/) and run `bash tools/init.sh` in the root folder to initialize the repo.  
4. Run the following commands in the root folder to install gems into `./vendor/bundle/` within the project—no need for **sudo** and no changes to `/var/lib/gems..`

```shell
bundle config set --local path 'vendor/bundle'
bundle install
```


## Usage

### Start the Jekyll Server

To run the site locally, use:

```shell
bundle exec jekyll s
```

### Configuration

Some variables to configure in `_config.yml` include:

* `lang`: set the website language
* `url`: your website URL
* `title`: main title shown under the avatar
* `tagline`: subtitle or site description
* `avatar`: supports local and CORS resources, including `gif`

### Comment feature via Giscus

We’ll use [**Giscus**](https://giscus.app) as the comment system. Other free alternatives include [**Disqus**](https://disqus.com/) and [**Utterances**](https://utteranc.es/).

**Steps**:

1. Install [**giscus**](https://github.com/apps/giscus) on GitHub.
2. In your repository **Settings**, go to **General** and enable **Discussions** so giscus can store comments.
3. Enter the repository name `<username>/<username>.github.io`. A green checkmark will appear when requirements are met.
4. Choose the discussion category and topic for your site.
5. Configure giscus in `_config.yml`:

   * `provider`: set to giscus
   * `giscus`: map the variables you set on [**giscus**](https://github.com/apps/giscus) to the `giscus` section

### Customize the Favicon

Create a custom **favicon** for your website instead of using the theme’s default.

**Steps**:

1. Go to [**Favicon Generator**](https://www.favicon-generator.org/).
2. Click <kbd>Browse</kbd> to select your favicon file, then <kbd>Create Favicon</kbd>.
3. Click `Download the generated favicon` to get the files.
4. Extract the downloaded file and delete these two:

   * browserconfig.xml
   * site.webmanifest
5. Copy the remaining files into `assets/img/favicons` (create the folder if it doesn’t exist).


## Extends

### Remove meta tag in Footer

You can remove the line `Using the Chirpy theme for Jekyll.` with these steps:

1. Create a file at `_data/locales/en.yml`.
2. Copy the contents of [**en.yml**](https://raw.githubusercontent.com/cotes2020/jekyll-theme-chirpy/refs/heads/master/_data/locales/en.yml) and paste into the new file.
3. Change the value of `meta` to `""`.

### Write a post

Check the [**rules**](https://chirpy.cotes.page/posts/write-a-new-post/) for writing blog posts with this theme and use [**Live Preview**](https://markdownlivepreview.com/) to preview your content.
