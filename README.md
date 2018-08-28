# How to Auto-Deploy Bikeshed Specs
Domenic Denicola has written a nice gist on [how to auto-deploy built products](https://gist.github.com/domenic/ec8b0fc8ab45f39403dd) to gh-pages using Travis.  You should read that first and the comments, because the gist is missing a few items.  The gist is fairly generic; we focus here on how to use it to automatically generate the HTML version of your spec using [bikeshed](https://github.com/tabatkins/bikeshed).


## Initial Setup
If you’re afraid of screwing up an existing repository, be sure to create a new fork of the repository and do all of the following steps there instead of the official repository.  When that’s working, you can copy over the necessary files to the official repository, and generate a new deploy key for the repository.

First, if you don’t have a master branch (many bikeshed specs have just the gh-pages branch), create one.  All of the following steps should be done on the master branch.

1. `git checkout -b master` (if needed)

## Create a Compile Script
This script, named compile.sh, will do everything needed to convert your index.bs to index.html, including any images and other ancillary files needed by the HTML version of your spec.

1. Create a script named `compile.sh`
1. Make it executable: `chmod a+x compile.sh`
1. Add it to your repo: `git add compile.sh`

Here is an example from https://github.com/wicg/cookie-store:
```bash
#!/bin/bash

# So we can see what we're doing
set -x

# Exit with nonzero exit code if anything fails
set -e

# Run bikeshed.  If there are errors, exit with a non-zero code
bikeshed --print=plain -f spec

# The out directory should contain everything needed to produce the
# HTML version of the spec.  Copy things there if the directory exists.

if [ -d out ]; then
    cp index.html out
fi
```

This script assumes that bikeshed produces no errors when converting the spec.  It also assumes that you only need index.html for your spec (no images, css, etc.). Copy everything you need to the out directory.

If your spec actually has errors you may need a more complicated script that compares the actual bikeshed errors with an expected list of errors.  See https://github.com/WebAudio/web-audio-api/blob/master/compile.sh for one possibility.

## Create a Deploy Script
Here you can basically copy the script from [the gist](https://gist.github.com/domenic/ec8b0fc8ab45f39403dd), or use the one from [cookie-store](https://github.com/WICG/cookie-store/blob/master/deploy.sh) or [web audio](https://github.com/WebAudio/web-audio-api/blob/master/deploy.sh). Name it `deploy.sh`. Note that, since bikeshed always appears to change `index.html` on every run, you’ll end up deploying a new `index.html` for any change you make on the master branch.

1. Create a script named `deploy.sh`
1. Add it to your repo: `git add deploy.sh`

## Create Your `.travis.yml` File
It should look something like this:
```yml
language: python
sudo: false
python:
  - "2.7"

install:
  # Setup bikeshed. See https://tabatkins.github.io/bikeshed/#install-linux
  - git clone https://github.com/tabatkins/bikeshed.git
  - pip install --editable $PWD/bikeshed
  - bikeshed update
 
script:
  - bash ./deploy.sh

env:
  global:
  - ENCRYPTION_LABEL: "label"
  - COMMIT_AUTHOR_EMAIL: "author-email"
```
This sets up Travis to include python 2.7 so we can run bikeshed.  The install step creates a clone of the latest version of bikeshed.  You may want to checkout a certain version of bikeshed; you can do that after cloning and before running pip:  git checkout <version>

1. Create a file named `.travis.yml`
1. Add it to your repo: `git add .travis.yml`

The `COMMIT_AUTHOR_EMAIL` is the email address that git will use when committing changes.  Any address can be used here, but it’s probably nice to have it be a real address such as the working group address.

The `ENCRYPTION_LABEL` will be set up below.

## Setting Up Travis
Follow the directions in [the gist](https://gist.github.com/domenic/ec8b0fc8ab45f39403dd) for setting up Travis. 

1. Go to https://travis-ci.org/
1. Next to **My repositories**, click the **+** button
1. Click on your username or the organization (e.g. WHATWG, WICG, etc) that owns the repo.
1. Find the repo in the list, and click the switch to enable Travis for it.

You will also need the travis client installed locally. On linux, you can do:

1. `sudo apt install ruby ruby-dev`
1. `sudo gem install travis`

## Create Encrypted Credentials
Follow the instructions in the gist for getting encrypted credentials, summarized here:

1. [Generate a new SSH key](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).  
    1. `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`
    1. At the **Enter a file...** prompt, use: `deploy_key`
    1. At the passphrase prompts, just hit Enter for no passphrase (twice).
1. Add the public key (in `deploy_key.pub`) as a deploy key in your repository:
    1. Go to `https://github.com/<proj-name>/<repo-name>/settings/keys` (You MUST be an admin to do this, or have an admin do these steps for you.)
    1. Choose an appropriate title, e.g. "Travis Bikeshed Build Deploy"
    1. Paste the contents of `deploy_key.pub`
    1. Check the option to allow write access (if you don’t, Travis won’t be able to update the gh-pages branch.)
    1. Delete the local file: `rm deploy_key.pub` (no longer needed)
1. Use the Travis client to encrypt the deploy key
    1. `travis login --org` (and log in with github credentials, including 2FA)
    1. `travis encrypt-file deploy_key`
    1. You don’t need to do any of the suggested items, but make a note of the openssl comment; it contains the encryption label that you need to add to `.travis.yml`.  For example, you might see something like:
```
    openssl aes-256-cbc -K $encrypted_0a6446eb3ae3_key -iv $encrypted_0a6446eb3ae3_key ...
```
    1. Add the encrypted file to the repo: `git add deploy_key.enc`
    1. Delete the key: `rm deploy_key` (don't check it in!)
1. Update `.travis.yml` by setting `ENCRYPTION_LABEL` with the encryption label printed when you encrypted the deploy key above. In the example, the encryption label from the openssl command is `0a6446eb3ae3`.

## Finishing Up

As a sanity check, run `git status` &mdash; you should have these new files staged in the `master` branch:

* `.travis.yml`
* `compile.sh`
* `deploy.sh`
* `deploy_key.enc`

1. Commit these changes: `git commit -a -m "Prepare master branch for automatic build and deploy"`
1. Publish the changes: `git push -f origin master`

If it doesn't already exist, set up a `gh-pages` branch. (The `deploy.sh` script will create one automatically, but doing it manually here will let us get everything set up correctly on GitHub in the following steps.)

1. `git checkout -b gh-pages`
1. `git push -f origin gh-pages`

Now you need to change your default branch and set up GitHub pages.

1. Go to the repo's settings on GitHub: `https://github.com/<proj-name>/<repo-name>/settings`.
1. On the **Options** page, scroll down to find **GitHub Pages** section and set the source to be "gh-pages branch".
1. On the **Branches** page, set the default branch to "master".

Next, checkout your `gh-pages` branch, and change `.travis.yml` to be:

```yml
# For the gh-pages branch, we don't need anything special except to
# have travis run.  That's what the branches: section is for.

language: generic
branches:
  only:
    - gh-pages
```

## Test It Out
At this point, everything should be setup and when you make a change on the master branch and push it to your repository, Travis should run and automatically deploy index.html to the gh-pages branch.

## Bonus Points
Now would be a good time to remove all of the unneeded files:

* `master` branch:
   * Remove `index.html` if present
* `gh-pages` branch:
   * Remove `index.bs` if present

If you have a `README.md` file, you may want to [add a badge](https://docs.travis-ci.com/user/status-images/) to show the status of the build.  To check your builds, visit `https://travis-ci.org/<project>/<repo>`.  There will be a badge showing the status of the build.  Click it to get a dialog box that has options on what format you want to use.  Select markdown if you want to add it to your `README.md` file.  Copy the link and place it in your `README.md` file at an appropriate place.  This also makes it easy to navigate directly to builds to see what’s happening or why the build is not passing.

## Debugging
Of course, nothing ever goes as planned, so you might need to figure out what’s happening.  The `compile.sh` script includes `set -x` to show each command that is run.  You can do the same for `deploy.sh` to see why it’s failing.
