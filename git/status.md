Git: Status of Working Copy/Index
=================================

Programatically getting the status:

    # `__ path` for index/working copy status of each path
    git status --porcelain     

    # Check for changes staged to index (ignoreg unstaged working copy changes)
    git diff-index --quiet --cached HEAD --

Information about programatically getting the status of the working
copy and index is available in StackOverflow post
and blog post [

### References:

* [Checking for a dirty index or untracked files with Git](https://stackoverflow.com/q/2657935/107294) (second answer is higher voted than accepted answer)
* [Adding Git Status Information to your Terminal Prompt](http://0xfe.blogspot.jp/2010/04/adding-git-status-information-to-your.html)