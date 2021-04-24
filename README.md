# The Sketching Developer

This site is my attempt at explaining developer topics in a way that I would have understood them.

## Development

```shell
bundle install --path vendor/bundle
bundle exec jekyll serve
```

## Resizing sketchnote images

```shell
sips -Z 640 <input> --out <output>
```


### Problem installing racc

```
Fetching racc 1.5.2
Installing racc 1.5.2 with native extensions
Errno::EACCES: Permission denied @ rb_sysopen - /Users/lucas/sketchingdev.github.io/vendor/bundle/ruby/2.6.0/gems/racc-1.5.2/COPYING
An error occurred while installing racc (1.5.2), and Bundler cannot continue.
Make sure that `gem install racc -v '1.5.2' --source 'https://rubygems.org/'` succeeds before bundling.
```

I resolved this by:
1. [Installing rbenv](https://stackoverflow.com/a/53388305)
2. I didn't have permissions to write to the `./vendor/bundle/ruby/2.6.0/gems/racc-1.5.2` bundle, so I ran:
   ```shell
   cd vendor/bundle/
   sudo chmod -R g+rw *
   ```
3. I could then run the `bundle install` command above


## Thanks

  * [Jekyll][jekyll] - Static website generator
  * [Sparrow Theme][sparrow-theme] - Theme used by the site


[jekyll]: https://jekyllrb.com/
[sparrow-theme]: https://github.com/lingxz/sparrow
