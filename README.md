# sphinx-doc

This repository holds the sources for a User and Developer Documentation (in German).

In order to make use of the (modified) Makefile you first have to clone this repo,
and second, make a clone of oa-deepreen.github.io next to 'sphinx-doc' into gh-pages,
with the folder name 'html', i.e.

    [...]$ git clone git@github.com:OA-DeepGreen/sphinx-doc
    [...]$ mkdir gh-pages
    [...]$ cd gh-pages
    [.../gh-pages]$ git clone git@github.com:OA-DeepGreen/oa-deepgreen.github.io html

Then, in the *local* 'sphinx-doc', the command

    [.../sphinx-doc]$ make html

builds new `.html`-pages into `../gh-pages/html`.  

In order to build *and* upload changes use

    [.../sphinx-doc]$ make buildcommithtml

This will invoke `sphinx-build` (as above), and will do additionally `git commit; git push` 
inside `../gh-pages/html` in one go.
