---
layout: post
title: "Using a Forked Jekyll Theme for Github Pages"
date: 2020-09-21
tags: jekyll blog git github-pages
---

## Switch to Jekyll

I have abandoned blogging for a very long time. I used to build my blog on [Hexo](https://github.com/hexojs/hexo) and hosted it on Github Pages. A major drawback of this solution is that we have to first generate all static files locally, and then push all those files to the remote repository. As for [Jekyll](https://github.com/jekyll/jekyll), since it is officially supported by Github Pages, we only need to push sources to the remote repository. Github Pages will generate the static site automatically, and those static files are not a part of our repository, making our repo much tidier.

There is a great [guide](https://docs.github.com/en/github/working-with-github-pages/setting-up-a-github-pages-site-with-jekyll) on blogging with Github Pages and Jekyll, but this solution also has a pain point. Although Jekyll has plenty of plugins developed by the community, Github Pages only supports some [specific ones](https://pages.github.com/versions/), let alone most supported plugins are of an older version without newest features. If we want to beautify or add features to the blog, we have to modify corresponding templates by ourselves. Of course, we can add those modified templates to our Github Pages repo directly, but what if we want to use them as a separate theme like a Ruby gem. If this new theme is a fork of an existing theme, we might also want to update it from the upstream. Although Github Pages only supports specific Jekyll plugins, it could utilize any Jekyll theme that is hosted on Github. Therefore, we need to create our own Jekyll theme and host it on Github, then specify it as the the theme that should be used by Github Pages.

## Configuration

I forked the [minima](https://github.com/jekyll/minima) theme and plan to develop my own theme based on it. Give the forked repo a new name, better to be the name you want for this new theme, e.g. `minima-graysonliu`. Then, in `*.gemspec`, simply change the value of `spec.name` to `minima-graysonliu`. We now have a Github-hosted Jekyll theme called `minima-graysonliu`. To use this theme, navigate to our Github Pages repo, in `_config.yml`, delete the following line:
```yaml
theme: ...
```
and add a new line:
```yaml
remote_theme: graysonliu/minima-graysonliu
```
The value of the `remote_theme` is just the name of your new theme repo hosted on Github, and we are done.

Other changes are needed if we want to modify and test our new theme locally. We first clone both the blog (Github Pages) repo and the theme repo to our local disk. In `Gemfile` of the blog repo, change
```ruby
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima"
```
to
```ruby
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima-graysonliu", :path => "/LOCAL_PATH_TO_THEME_REPO/minima-graysonliu"
```
Update with:
```bash
bundle update
```
Even if `_config.yml` in the blog repo does not have the key `theme` (because we replaced it with `remote_theme`), since we specify `minima-graysonliu` as the default theme, Jekyll will use it when we run the server locally.

If we need to keep our forked new theme updated with the original theme, just set the original repo as its [upstream](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/configuring-a-remote-for-a-fork) and [sync](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/syncing-a-fork) the fork regularly.