# Blog

 [![GitHub last commit](https://img.shields.io/github/last-commit/vikaspogu/blog?color=purple&style=flat-square)](https://github.com/vikaspogu/blog/commits/master)

## Overview

Welcome to my blog repo

---

## :wrench:&nbsp; Tools

_Below are the tools in use in this repo_

| Tool                                                   | Purpose                                                                                                   |
|--------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| [go-task](https://github.com/go-task/task)             | Replacement for make and makefiles, who honestly likes that?                                              |

## Useful Information

### Cloning repo

```
git clone https://github.com/Vikaspogu/blog.git
cd blog
git rm --cached public -fr
git submodule add --force -b master git@github.com:vikaspogu/vikaspogu.github.io.git public
hugo
./deploy.sh "<commit message>"
```

### Update submodule

```
git submodule update --init --recursive
```

### Helpful Links

[Hugo site on GitHub Pages](https://dev.to/dgavlock/creating-a-hugo-site-on-github-pages-3cjo)
