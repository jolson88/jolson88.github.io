# Coding Coda
This is the personal blog of Jason Olson. It is powered by Jekyll (hosted on Github Pages), and currently using the [Minimal Mistakes](http://mmistakes.github.io/minimal-mistakes) theme.

## Usage
For the correct image and link generation, there is a separate _config.yml and _config-dev.yml (so certain links will hit localhost instead of the production url). To generate and serve up the current site locally, run:

```
bundle exec jekyll serve --config _config.yml,_config-dev.yml
```

As we have included the `github-pages` gem in our `Gemfile`, we use bundle exec here to ensure the right verison of jekyll is used. While jekyll 3.0 has been released, Github Pages currently only supports Jekyll 2.0 and below. So we don't want to accidentally use features locally that aren't supported once deployed to Github Pages.
