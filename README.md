# Blog

## This is my personal blog

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

### Links

[Hugo site on GitHub Pages](https://dev.to/dgavlock/creating-a-hugo-site-on-github-pages-3cjo)
