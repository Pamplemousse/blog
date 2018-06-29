---
layout: post
title: Patch option in Git
tags: [ git, development ]
---

## Patch

I have (almost) always been using `git add -p` or `git add --patch`.

This option allows you to interactively select which pieces of your changes to be added to the index.
(Before writing this article, I was even convinced that `-p` stood for "partial"...)

This is very convenient to 1) make sure that you will not commit unwanted code, 2) partially save your changes without losing your work in progress.

I recently learned that this `-p`/`--patch` option is available for the [`checkout`](https://git-scm.com/docs/git-checkout) and the [`reset`](https://git-scm.com/docs/git-reset) commands as well!

With these, we can respectively get rid of only part of our changes and remove pieces of code from the index.


## Examples

Here is the example setup: I created a git repository in which I have committed a single file.

```bash
$ ls
example.txt

$ git status
On branch master
nothing to commit, working tree clean

$ cat example.txt
This is an exmple fil.
Containing multiple lines.

Very interest.
```

### git add -p

Let's say we edited our `example.txt` to add some content that we would like to commit.

```bash
$ cat example.txt
File

This is an example file.
Containing multiple lines.

Very interesting.
```

We did multiple things here: added a "title" an corrected several words.
To keep things clean, we would like to make one commit for each one of these changes.

Here is how I would use `-p`:
  * use "split" and "edit" to keep only the changes related to correct the words
  * commit those changes
  * verify that only the title is added
  * commit this change


<img alt="showing the use of `git add -p`" src="/assets/images/2018-06-29-Patch%20option%20in%20Git/add_patch.gif">


### git checkout -p

Similarly, we can use `git checkout -p` to discard part of the changes that we have performed on a file.
Let's say we have edited our `example.txt` to add a line in the middle and modify the last one.

```bash
$ cat example.txt
File

This is an example file.
Bwaaaaaaaaaah!
Containing multiple lines.

Some very interesting changes.
```

Then, we can use `git checkout -p` to get rid of the rubbish line that has been introduced:
  * use split
  * get rid of the first part
  * but not of the second

<img alt="showing the use of `git checkout -p`" src="/assets/images/2018-06-29-Patch%20option%20in%20Git/checkout_patch.gif">


### git reset -p

At last, we added our previous change to the index, as well as our edit of the third line, containing a grammar error...

```bash
$ cat example.txt
File

This is a example marvelous file.
Containing multiple lines.

Some very interesting changes.

$ git add example.txt
```

We changed our mind: this is not OK to commit broken English.

Let's use `git reset -p` to remove the unwanted content from the index so we can commit peacefully:
  * split the content
  * reset the first part
  * keep the second one
  * commit

<img alt="showing the use of `git reset -p`" src="/assets/images/2018-06-29-Patch%20option%20in%20Git/reset_patch.gif">


**Et voilà!**
