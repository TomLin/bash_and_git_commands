
## Git Work Flow

* create new branch: `git branch <new_branch_name>` VS delete branch: `git branch -d <new_branch_name>`
* list all existing branches: `git branch --list`
* switch to branch: `git checkout <new_branch_name>` VS simultaneously creates and checkout to new branch: `git checkout -b <new_branch_name>`
* add new file: `git add <file_name>`
* commit: git `git commit -m "message"`
* create a branch on remote repository: `git push --set-upstream origin <new_branch_name>` Another version: `git push -u origin <new_branch_name>`
* update the local repository: `git pull`
* merge the brnach/master: `git merge <branch_name>`

## Relatively Unimportant Git Command
* see the remote url: `git remote -v`
* list all branches on remote repository: `git branch -r` and exit the reading mode: `enter q`
* revert the file in staging area back to working directory: `git reset HEAD <file_name>`
* revert to the original commit: `git reset HEAD~1`
* revert the changed file back to its original version in working directory: `git checkout HEAD <file_name>`
* `git fetch` and `git pull` both have similar purpose


### Error Message Case01

[webpage](https://mrvirk.com/blog/2018/01/27/solution-githubgitlab-please-enter-a-commit-message-to-explain-why-this-merge-is-necessaryespecially-if-it-merges-an-updated-upstream-into-a-topic-branch-error-message/)

When trying to push a commit into Github via terminal we come across this problem:

**“Please enter a commit message to explain why this merge is necessary,especially if it merges an updated upstream into a topic branch | Error Message”**

This happens when a commit was made to the branch you are working with an you try to push a commit (git push) before pulling the changes (git pull)

How to solve this:

1. Press “i” on your keyboard.
2. Write your merge message
3. Press “esc” button
4. Type “:wq”
5. Press Enter
6. Finally, Push Changes “git push”

