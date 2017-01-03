# SOSS-SIG Web Source

## How to install
(install hugo)

`$ git clone git@github.com:jsosug/sossweb.git`

`$ cd sossweb`

`$ git clone git@github.com:jsosug/jsosug.github.io.git public`

## How to write blog post
`$ hugo new post/{{slug-name}}.md`

`$ #edit {{slug-name}}.md`

## How to deploy
`hugo --theme=liquorice #update public/`

`$ cd public`

`$ git commit`

`$ git push origin master #deploy to github.io`