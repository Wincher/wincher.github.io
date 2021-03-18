# WincherToldYou

## HEXO install  

`npm install hexo -g`  
`hexo -v`  

## plugin:generate relative path for image

`npm install hexo-asset-image --save`

git clone git@github.com:Wincher/Wincher.github.io.git
git checkout hexo

~~themes/_material_src is used to linked in submodule~~
~~link all source to material exactly as the path tree~~
~~this may occur some unknown issue, stuck when run hexo server~~
themes/_material_src is used to copy to submodule
copy all sources to material exactly as the path tree

```bash
#eg:
cp ../_material_src/source/img/avatar.jpeg  source/img/avatar.jpeg
```

## deploy operation

`git pull`  
`git add .`  
`git commit -m'comment'`  
`git push orign hexo` this is used for backup posts in hexo branch 
`hexo g -d` this is used for generate souce and push generated sources to master 

## other

`hexo init`  
`hexo g`
`hexo s`

## create a new post

`hexo new`

