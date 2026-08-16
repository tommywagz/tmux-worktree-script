# tmux-worktree-script
Local bash script to launch configurable git worktree branches with tmux. You can put this script
wherever you want to in your file directory. ~/.local/bin/ if you want to be able to run it anywhere
in your machine's local directory. or ust place it within a specific project folder that you want
to launch multiple tmux windows in. 

# How to Use the Script
There are a few input parameters that the script requires upon being called. This includes the 
session name, the number of worktrees that you want to be working in parallel, along with the name
of your tmux session and the name of the base git branch that you want to branch off of with 
your worktree. The number of worktree branches and thus tmux panes defaults to 3 and the 
base_branch name defaults to 'main'.

Here is an example of a standard use creating 3 worktree branches off of the git branch 'main'
in a session newly titled 'new-feature':
<mandatory>
[optional]

work <session-name> [num_worktrees] [base_branch]

work new-feature 4 master

## Cleanup
The script comes with a '--clean' tag where you can kill a given session:

work --clean <session-name>

work --clean new-feature
