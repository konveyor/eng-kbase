# Konveyor Engineering Knowledgebase

[![Netlify Status](https://api.netlify.com/api/v1/badges/4be93a33-6977-4964-ba22-fc1edf0c0fa6/deploy-status)](https://app.netlify.com/sites/konveyor-eng-kbase/deploys)

This repo is a dedicated place for contributors to the Konveyor project to post
general engineering related information of all kinds. This could be everything
from technical deep dives to tips and tricks. All contributors welcome, please
propose new posts in the form of a PR.

## Writing a new post

To start a new post, use the following command:

`hugo new posts/my-new-post.md`

Be sure to include hyphens in your file name as we have templates set up that
will follow these conventions. You should automatically get the time string
included in your post.

### Categorizing and Tagging

Be sure to add your `category` and `tags` to your post metadata so that it can
be discovered by folks who are browsing the knowledgebase.

## Updating the theme

Instead of using submodules, we're just committing the theme directly and will
manually perform updates as they're not expected to change often and submodules
are a pain.

`git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1`

`rm -rf themes/PaperMod`, reclone to update, `rm -rf themes/PaperMod/.git`, commit and PR.
