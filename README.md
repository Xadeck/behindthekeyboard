Behind The Keyboard
====

This GitHub project contains my personal blog, as an Octopress directory which is then deployed on GiHub pages.

# For external contributors

If you want to edit the content of the blog, you can clone this project:
```bash
git clone https://github.com/Xadeck/behindthekeyboard.git
cd behindthekeyboard/
bundle update
```
Then edit the `source/_posts` and preview locally the change:
```bash
rake preview
```
and browse [http://localhost:4000](http://localhost:4000). Submit your proposed change via a [pull request](https://help.github.com/articles/using-pull-requests).

# For internal contributors

## Editing posts

Just clone the project, and link the `_deploy` directory to the GitHub pages one (you must have access to this is a private repository):

```bash
git clone https://github.com/Xadeck/behindthekeyboard.git
cd behindthekeyboard/
bundle update
git clone https://github.com/Xadeck/xadeck.github.io.git _deploy
```

Then do your edit, and simply deploy them:
``` bash
rake generate
rake deploy
```
## Updating Octopress

Add once Octopress's remote repository to your clone:
``` bash
cd behindthekeyboard/
git remote add octopress git://github.com/imathis/octopress.git
```

Then update from Octopress's master whenever desired using [Updating Octopress](http://octopress.org/docs/updating/) instructions. Short version is :
``` bash
git pull octopress master     # Get the latest Octopress
bundle install                # Keep gems updated
rake update_source            # update the template's source
rake update_style             # update the template's style
```



