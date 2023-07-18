# Howto install, generate and deploy

See guide [here](https://help.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll) for more info.

## Steps

- Install ruby 2.7 {Ruby+Devkit 2.7.8-1 (x64)}

  `goto https://rubyinstaller.org/downloads/`

- Install ruby bundler

  `$ gem install bundler -g`

- Develop and test locally

  `$ bundler exec jekyll serve`

- Git commit & push

  `$ git commit --all`
  `$ git commit -am "change: some commit message"`
  `$ git push`
