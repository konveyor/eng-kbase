# eng-kbase
Konveyor Engineering Knowledgebase

## Updating the theme

Instead of using submodules, we're just committing the theme directly and will
manually perform updates as they're not expected to change often and submodules
are a pain.

`git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1`

`rm -rf themes/PaperMod`, reclone to update, `rm -rf themes/PaperMod/.git`, commit and PR.
