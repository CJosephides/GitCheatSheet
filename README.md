# Git Cheat Sheet

This is a little cheat sheet that I've put together to remind myself of some common git operations when I get rusty.

## Committing

* Each commit keeps a reference to its parent(s).

## Branching

* A branch is simply a pointer to a specific commit.
* ```git checkout -b <branch_name>``` is a shortcut for
```
git branch <branch_name>  # create a new branch
git checkout <branch_name>  # switch to it
```
* Branch forcing is the action of reassigning a branch, which is simply a reference, to a different commit. For example
```
git branch -f master HEAD~3
```
will force the master branch to three commits behind HEAD.
* *Remember that we can't force `HEAD` around like this. We should just use `checkout <some_ref>` to detach and move `HEAD`.*

## Merging

* `git merge` will create a new commit that has two (or more?) parents.
* Suppose that we are on branch `master`; then, the command `git merge bugFix` will merge branch `bugFix` into `master`.
* To distinguish from `rebase`, I like to read `merge <ref>` as: "merge [ref] into my stuff here".

## Rebasing

* Rebasing is another way of combining work between branches. Rebasing copies a set of commits somewhere else. Its main advantage over simple `merge` is that it can be used to create a linear sequence of commits, thus keeping the commit history clean.
* Suppose we are on branch `bugFix` and we want to move our work onto the `master` branch. Rather than use `merge`, which would reveal the fact that the two branches were developed independently, we can use rebase. We do this with
```
git rebase master
```

For example:

```
             c_0             
              |
             c_1
            /   \
 (master)c_2     c_3(bugFix)*
```

becomes

```
             c_0             
              |
             c_1
            /   \
 (master)c_2     c_3
          |
(bugFix*)c_3'
```

We'll want to move `master` down to the latest commit with

```
git checkout master
git rebase bugFix
```

There is nothing complicated to do here: since `master` was an ancestor of `bugFix`, git will simply move the `master` pointer to `c_3'`. We end up with:

```
             c_0             
              |
             c_1
            /   \
         c_2     c_3
          |
(bugFix) c_3'
(master*)
```


* *Note: unlike `merge`, `rebase` moves the contents of the **curent** branch to the destination. I like to read `rebase <ref>` as: "rebase my stuff onto [ref]".*

## Moving around

* `HEAD` is a symbolic name for the currently checked-out commit. Normally, `HEAD` points to a branch name.
* We can detach `HEAD`, and have it point to a commit instead of a branch by doing `git checkout <commit_hash>`.

### Relative movement and commits

* Move upwards one commit with `^`.
* Move upwards a number of commits with `~<num>`.
* Append the caret and tilde operators to a reference name to move relative to that reference.
* If our (`HEAD`) reference has multiple parents, we can append a number after the caret operator to specify the parent we want. For example:

```
    c_0
   /   \
c_1     c_2
   \   /
    c_3 (master)
```

doing `git checkout master^` will pick the left-most parent; doing `git checkout master^2`, however, will pick the second-from-the-left parent.
* We can chain these operators; for example:

```
git branch bugFix HEAD^^2~2
```

will move one up from `HEAD`, then move up again to the second parent, then move up another two commits, and create branch `bugFix` there.

## Reversing changes

* `git reset <ref>` will move `HEAD` (i.e. the current branch it is pointing to) backwards, as if the commit had never happened.
* `git reset` works fine for local branches but it is not intended for sharing the reversion with others on remote branches. For that, we need to use `git revert`.
* `git revert HEAD` will make a *new* commit down from `HEAD` that will contain changes that exactly reverse the current commit's changes relative to its parent. We can now push this new commit to our remotes.

## Moving work around

### Cherry-pick

* `git cherry-pick` allows us to copy a set of commits below our `HEAD`. For example, doing

```
git cherry-pick c3 c4 c7
```

while `HEAD` is on `master`, will copy `c3, c4, c7`, in that order, below `master`. This is useful for mixing and matching commits on different branches.

* A common scenario arises when we branch off the development branch to fix a bug. Suppose our debugging work involves a series of commits that add debug and print statements. Finally, we find the fix and introduce it in the latest commit. Our commit history looks like:

```
c_0
 |
c_1 (master)
 |
c_2
 |
c_3
 |
c_4 (bugFix) (HEAD)
```

We want to "copy" only the work from `c_4` onto master. We can use cherry-pick for this as follows:

```
git checkout master
git cherry-pick bugFix
```

This will result in

```
    c_0
     |
    c_1 
   /   \
c_2     c_5 (master) (HEAD)
 |
c_3
 |
c_4 (bugFix)
```

*The other way to do this is through an interactive rebase.*

* The other syntax form for `rebase` lets us work with any two commits, without having to have HEAD on either, but it is a bit backward:

```
git rebase master bugFix
```

will copy branch `bugFix` below `master`.

### Interactive rebase

* Use `git rebase -i` for interactive rebasing. It is a good way to review a series of commits we are about to rebase.
* Use `git rebase -i HEAD~4`, for example, to interactively rebase the current `HEAD` onto four commits back.
* When the commit history is linear, this allows us to re-order and/or ommit commits. For example, suppose we have:

```
c_0
 |
c_1
 |
c_2
 |
c_3
 |
c_4
 |
c_5 (master) (HEAD)
```

With `git rebase -i HEAD~4` we are now asking: "what changes from `c_2` to `c_5`, inclusive, do I want to copy on top of `c_1`; and, in what order?" We could, for example, end up with something like:

```
     c_0
      |
     c_1
    /   \
 c_2     c_3'
  |       |
 c_3     c_5'
  |       |
 c_4     c_4' (master) (HEAD)
  |
 c_5
```

where we have ommitted `c_2` and re-ordered `c_3` through `c_5`.

* *Remember that `rebase` is read "rebase my stuff onto <ref>" and moves HEAD onto the rebased side.*
* A nice use-case for interactive rebase is when we want to make a small amendement to a previous commit. An idiom we can use is as follows:
    1. Do an interactive rebase one commit *prior* to the commit we wish to amend. Re-order the commits such that the to-be-amended commit is the latest one.
    2. Do a `git commit --amend`.
    3. Do a second interactive rebase to put things back in their original order.
    4. If necessary, move branch references around (by forcing).

*We could also accomplish this through `git cherry-pick`, without having to rebase stuff around.*

## Tags

* Branches are mutable during the lifetime of a project. They cannot be relied upon to act as permanent references to some commit.
* Tags provide a way to mark some commits permanently. We can think of tags are milestones. We can't move or checkout tags.
* The syntax for taggins is

```
git tag <tag_name> <ref_name>
```

where `<ref_name>` is some commit hash or (relative) reference. In the absence of a `<ref_name>` git will implicitly use `HEAD`.

* Use `git describe` to ask git where we are relative to the closest tag. The syntax is

```
git describe [ref_name]  # optional ref_name, otherwise HEAD
```

The output reads like:

```
<the_closest_tag>_<is_number_of_commits_away>_<from_commit_hash_for_ref_name>
```

## Remotes

* Remote branches have the `<remote_name>/<branch_name>` naming convention.
* Remote branches reflect the state of remote repositories. When we check out a remote branch, we place `HEAD` into a detached state. This is done on purpose to enforce that we can't work on these branches directly. For example:

```
             [local]   [remote:origin]
               c_0           c_0
(HEAD)          |             |
(master)       c_1           c_1 (master)
(origin/master)
```

Doing `git checkout origin/master` will detach HEAD and point it to `origin/master`. If we then commit, we end up with:

```
             [local]   [remote:origin]
               c_0           c_0
                |             |
(master)       c_1           c_1 (master)
(origin/master) |
               c_2 (HEAD)
```

`origin/master` will only update when the remote updates. We can't affect the remote state unless we push.

### Fetch

* `git fetch` performs two steps:
    1. Downloads the commits that the remote has but are missing from our local repository.
    2. Updates where our local remote branches point.
It essentially brings our local representation of the remote into synchronization with the remote repository. It will not, however, change anything related to our local branches.
* Running just `git fetch` will update all remote branches.

### Pull

* Once we have downloaded remote commits with `git fetch` we are responsible for updating our local branches to incorporate those changes. We can do this via the usual methods of `cherry-pick`, `rebase`, and `merge`. More commonly, however, we can simply use `git pull`.
* `git pull` executes a `fetch` and a `merge` sequentially. For example, suppose we have:

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
(origin/master) c_1          c_1 
                 |            |
(HEAD) (master) c_2          c_3 (master)
```

Then, doing `git pull origin master` will fetch `c_3` from the remote and then merge it with our `master`. The commit history will look like:

```
              [local]                    [remote:origin]
                c_0                            c_0
                 |                              |
                c_1                            c_1 
               /   \                            |
            c_2     c_3 (origin/master)        c_3 (master)
               \   /
                c_4 (master)
```

Note how the `origin/master` branch stays with `c_3`.

### Push

* `git push` will publish our changes to the remote. It will also update the remote's branches to point to what we just pushed. Because the remote's branches are updated, our local remote branches will also be updated. For example, suppose we have:

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
(origin/master) c_1          c_1 (master)
                 |           
(HEAD) (master) c_2           
```

After `git push origin master` the state of our local and remote repositories will be:

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
                c_1          c_1 
                 |            |
(HEAD) (master) c_2          c_2 (master)
(origin/master)
```

### Divergence

Consider the following situation. We have clone a repository and work on some feature. By the time we are done, however, the repository has been updated by other people in such a way that makes our work irrelevant. Git will not allow us to push in this case; rather, it first forces to incorporate the latest state of the remote before being able to share our work. For example:

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
(origin/master) c_1          c_1 
                 |            |
(HEAD) (master) c_3          c_2 (master)
```

`git push origin master` will fail here, because our local representation of the remote's `master` is behind. Because our commit has this (stale) ancestor, git will not allow us to commit since our repositories have diverged.

A common way to resolve this is via fetching and rebasing before pushing. The state after

```
git fetch origin master
git rebase origin master  # recall HEAD is on c_3
```

will be

```
              [local]                 [remote:origin]
                c_0                         c_0
                 |                           |
                c_1                         c_1 
               /   \                         |
            c_3     c_2 (origin/master)     c_2 (master)
                     |
                    c_3 (master) (HEAD)
```

and after `git push origin master`

```
              [local]                 [remote:origin]
                c_0                         c_0
                 |                           |
                c_1                         c_1 
               /   \                         |
            c_3     c_2                     c_2
                     |                       |
                    c_3 (master) (HEAD)     c_3 (master)
                        (origin/master) 
```

*There is a convenient `git pull --rebase origin master` shortcut, which can be followed by `git push origin master` to achieve the above.*

We could also do this via `merge`, rather than `rebase`. The end state after

```
git fetch
git merge origin master
git push origin master
```

is the same, but the history looks a little different:

```
              [local]                 [remote:origin]
                c_0                         c_0
                 |                           |
                c_1                         c_1 
               /   \                       /   \              
            c_3     c_2                 c_2     c_3
               \   /                       \   /
                c_4 (master) (HEAD)         c_4 (master)
                    (origin/master) 
```

*A convenient shortcut here is simply `git pull origin master` followed by `git push origin master`. This will construct the same state and history as the merge solution above.*

### Rebase versus merge

* Rebasing tends to produce cleaner commits, which are more often linear; however, it modifies the true history of the repository.
* Merging, on the other hand, preserves the commit history.

### Remote-tracking branches

How does git know that our `master` branch is related to `origin/master`?

* When we cloned the remote, remote-tracking branches were created for all of the repository's current branches.
* Actually, the above may not be correct. It seems that git only created a local and remote-tracking branch for the remote's currently active branch (usually `master`).
* We can specify that a new branch tracks some branch on the remote through

```
git checkout -b totallyNotMaster origin/master
```

* We set an existing branch to track a remote branch through

```
git branch -u origin/master myBranch
```

### Push arguments

* From *Learn Git Branching*:

"""
git push origin master

translates to this in English:

Go to the branch named "master" in my repository, grab all the commits, and then go to the branch "master" on the remote named "origin". Place whatever commits are missing on that branch and then tell me when you're done.
"""

This can be useful when we don't want to move `HEAD` before pushing our branch to the remote.

* The syntax `git push origin <source>:<destination>` allows us to push local branch <source> to remote branch <destination> on the `origin` remote. Suppose we have a state:

```
              [local]  [remote:origin]
                c_0          c_0
(master)         |            |
(origin/master) c_1          c_1 (master)
                 |           
                c_2          
                 |
(HEAD) (foo)    c_3
```

After `git push origin foo^:master` we will have:

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
(master)        c_1          c_1
                 |            | 
(origin/master) c_2          c_2 (master)
                 |
(HEAD) (foo)    c_3
```

Note how the local `origin/master` and remote `master` branches have both been updated consistently.

In fact, the destination branch does not have to exist. If we were to instead say:

```
git push origin foo^:newBranch
```

we would get

```
              [local]  [remote:origin]
                c_0          c_0
                 |            |
(master)        c_1          c_1 
                 |            | 
(origin/master) c_2          c_2 (master)
(newBranch)      |               (newBranch)
(HEAD) (foo)    c_3
```

(I think)

### Fetch arguments

* These are very similar to the arguments of `git push` -- just in reverse.
* With the syntax `git fetch origin foo` we are asking git to go into the `origin` remote's `foo` branch, grab all the commits that aren't present locally, and copy them down onto the local `origin/foo` branch, which is tracking `foo`.
* Remember that `git fetch` never updates local *work* branches; instead, it always updates local *tracking* branches.
* Most importantly, `git fetch`, without any arguments, will download all the commits from the remote onto all the local remote-tracking branches.
* The syntax `git fetch origin <remote_branch>:<local_branch>` will fetch all changes from remote's <remote_branch> and put them into our local <local_branch>. It will create <local_branch> if it doesn't exist.

### Source of nothing

* The syntax `git push origin :foo` will actually delete the remote `foo` branch (by pushing nothing to it).
* The syntax `git fetch origin :bar` will create a new branch!

### Pull arguments

* Suppose we have the state

```
      [local]              [remote:origin]
        c_0                      c_0
         |                        |
        c_1 (master)             c_1
(HEAD)   |  (origin/master)       |
(bar)   c_2                      c_3 (master)
```

Doing `git pull origin master` will do a `fetch` (master) followed by a `merge` with whatever we happen to be at currently, resulting in:

```
      [local]              [remote:origin]
        c_0                      c_0
         |                        |
        c_1                      c_1
       /   \    (origin/master)   |
    c_2     c_3 (master)         c_3 (master)
       \   /
        c_4 (bar) (HEAD)
```

* We can use the syntax `git pull origin master:foo` when the local branch `foo` does not exist. This will create `foo` wherever `HEAD` happens to be currently pointing and download the remote's `master` into it. Then, it will merge `foo` into the branch pointed to by `HEAD` (leaving `foo` at the previous position of `HEAD`). For example, given:

```
              [local]   [remote:origin]
                c_0           c_0
       (master)  |             |
(origin/master) c_1           c_1
                 |             |
         (HEAD) c_3           c_2 (master)
          (bar)
```

after `git pull origin master:foo` we will be in:

```
  [local]
    c_0
     |
    c_1 (master) (origin/master)
   /   \
c_3     c_2 (foo)
   \   /
    c_4 (bar) (HEAD)
```

Importantly, note how our `origin/master` is *not* updated. 

# References

* [Learn Git Branching](https://learngitbranching.js.org)
