
# Setup

To serve up a local version of this site, first install ruby

```
pacman -S ruby
```

Then install jekyll

```
gem update
gem install jekyll
```

If those commands tell you to add a directory to your PATH, do that.

Then you can launch a local version of the site on localhost:4000 with

```
jekyll serve --drafts --future &>/dev/null &
```

