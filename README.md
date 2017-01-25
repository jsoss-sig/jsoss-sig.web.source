# SOSS-SIG Web Source

## How to install
(install hugo)

`$ git clone git@github.com:jsoss-sig/jsoss-sig.web.source.git`

`$ cd jsoss-sig.web.source`

`$ git clone git@github.com:jsoss-sig/jsoss-sig.github.io.git public`

`$ cd ../themes`

`$ git https://github.com/eliasson/liquorice.git liquorice`

`cd ../`

## How to write blog post
`$ git branch -b NEWTOPICNAME`

`$ hugo new post/{{slug-name}}.md`

`$ #edit {{slug-name}}.md`

## How to deploy
`hugo --theme=liquorice #update public/`

`$ cd public`

`$ git commit`

`$ git push origin master #deploy to github.io`

## How to share source of new articles

`cd jsoss-sig.web.source`

`git push origin NEWTOPICNAME`
