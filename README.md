# Vikaspogu.dev (Blog)

 [![GitHub last commit](https://img.shields.io/github/last-commit/vikaspogu/blog?color=purple&style=flat-square)](https://github.com/vikaspogu/blog/commits/master)
 ![SIZE](https://img.shields.io/github/repo-size/vikaspogu/blog?style=flat-square)

## Overview

Welcome to my blog repo based on [congo](https://jpanther.github.io/congo/)

---

## :wrench:&nbsp; Tools

_Below are the tools in use in this repo_

| Tool                                                   | Purpose                                                                                                   |
|--------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| [go-task](https://github.com/go-task/task)             | Replacement for make and makefile, who honestly likes that?                                              |

## Useful Information

### Cloning repo

```bash
git clone https://github.com/Vikaspogu/blog.git && cd blog
hugo server -D
rm -rf public
git rm --cached public -fr
git submodule add --force -b master git@github.com:vikaspogu/vikaspogu.github.io.git public
hugo
task push COMMIT_MSG="<commit message>"
```

### Update submodule

```bash
git submodule update --init --recursive
```

### Helpful Links

[Hugo site on GitHub Pages](https://dev.to/dgavlock/creating-a-hugo-site-on-github-pages-3cjo)
