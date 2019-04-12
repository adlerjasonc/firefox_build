bootstrap.py

```py
#!/usr/bin/env python
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this file,
# You can obtain one at http://mozilla.org/MPL/2.0/.

# This script provides one-line bootstrap support to configure systems to build
# the tree.
#
# The role of this script is to load the Python modules containing actual
# bootstrap support. It does this through various means, including fetching
# content from the upstream source repository.

# If we add unicode_literals, optparse breaks on Python 2.6.1 (which is needed
# to support OS X 10.6).

from __future__ import absolute_import, print_function

WRONG_PYTHON_VERSION_MESSAGE = '''
Bootstrap currently only runs on Python 2.7 or Python 2.6.
Please try re-running with python2.7 or python2.6.

If these aren't available on your system, you may need to install them.
Look for a "python2" or "python27" package in your package manager.
'''

import sys
if sys.version_info[:2] not in [(2, 6), (2, 7)]:
    print(WRONG_PYTHON_VERSION_MESSAGE)
    sys.exit(1)

import os
import shutil
from StringIO import StringIO
import tempfile
try:
    from urllib2 import urlopen
except ImportError:
    from urllib.request import urlopen
import zipfile

from optparse import OptionParser

# The next two variables define where in the repository the Python files
# reside. This is used to remotely download file content when it isn't
# available locally.
REPOSITORY_PATH_PREFIX = 'python/mozboot/'

TEMPDIR = None


def setup_proxy():
    # Some Linux environments define ALL_PROXY, which is a SOCKS proxy
    # intended for all protocols. Python doesn't currently automatically
    # detect this like it does for http_proxy and https_proxy.
    if 'ALL_PROXY' in os.environ and 'https_proxy' not in os.environ:
        os.environ['https_proxy'] = os.environ['ALL_PROXY']
    if 'ALL_PROXY' in os.environ and 'http_proxy' not in os.environ:
        os.environ['http_proxy'] = os.environ['ALL_PROXY']


def fetch_files(repo_url, repo_rev, repo_type):
    setup_proxy()
    repo_url = repo_url.rstrip('/')

    files = {}

    if repo_type == 'hgweb':
        url = repo_url + '/archive/%s.zip/python/mozboot' % repo_rev
        req = urlopen(url=url, timeout=30)
        data = StringIO(req.read())
        data.seek(0)
        zip = zipfile.ZipFile(data, 'r')
        for f in zip.infolist():
            # The paths are prefixed with the repo and revision name before the
            # directory name.
            offset = f.filename.find(REPOSITORY_PATH_PREFIX) + len(REPOSITORY_PATH_PREFIX)
            name = f.filename[offset:]

            # We only care about the Python modules.
            if not name.startswith('mozboot/'):
                continue

            files[name] = zip.read(f)
    else:
        raise NotImplementedError('Not sure how to handle repo type.', repo_type)

    return files


def ensure_environment(repo_url=None, repo_rev=None, repo_type=None):
    """Ensure we can load the Python modules necessary to perform bootstrap."""

    try:
        from mozboot.bootstrap import Bootstrapper
        return Bootstrapper
    except ImportError:
        # The first fallback is to assume we are running from a tree checkout
        # and have the files in a sibling directory.
        pardir = os.path.join(os.path.dirname(__file__), os.path.pardir)
        include = os.path.normpath(pardir)

        sys.path.append(include)
        try:
            from mozboot.bootstrap import Bootstrapper
            return Bootstrapper
        except ImportError:
            sys.path.pop()

            # The next fallback is to download the files from the source
            # repository.
            files = fetch_files(repo_url, repo_rev, repo_type)

            # Install them into a temporary location. They will be deleted
            # after this script has finished executing.
            global TEMPDIR
            TEMPDIR = tempfile.mkdtemp()

            for relpath in files.keys():
                destpath = os.path.join(TEMPDIR, relpath)
                destdir = os.path.dirname(destpath)

                if not os.path.exists(destdir):
                    os.makedirs(destdir)

                with open(destpath, 'wb') as fh:
                    fh.write(files[relpath])

            # This should always work.
            sys.path.append(TEMPDIR)
            from mozboot.bootstrap import Bootstrapper
            return Bootstrapper


def main(args):
    parser = OptionParser()
    parser.add_option('--vcs', dest='vcs',
                      default='hg',
                      help='VCS (hg or git) to use for downloading the source code. '
                      'Uses hg if omitted.')
    parser.add_option('-r', '--repo-url', dest='repo_url',
                      default='https://hg.mozilla.org/mozilla-central/',
                      help='Base URL of source control repository where bootstrap files can '
                      'be downloaded.')
    parser.add_option('--repo-rev', dest='repo_rev',
                      default='default',
                      help='Revision of files in repository to fetch')
    parser.add_option('--repo-type', dest='repo_type',
                      default='hgweb',
                      help='The type of the repository. This defines how we fetch file '
                      'content. Like --repo, you should not need to set this.')

    parser.add_option('--application-choice', dest='application_choice',
                      help='Pass in an application choice (see mozboot.bootstrap.APPLICATIONS) '
                      'instead of using the default interactive prompt.')
    parser.add_option('--no-interactive', dest='no_interactive', action='store_true',
                      help='Answer yes to any (Y/n) interactive prompts.')

    options, leftover = parser.parse_args(args)

    try:
        try:
            cls = ensure_environment(options.repo_url, options.repo_rev,
                                     options.repo_type)
        except Exception as e:
            print('Could not load the bootstrap Python environment.\n')
            print('This should never happen. Consider filing a bug.\n')
            print('\n')
            print(e)
            return 1
        dasboot = cls(choice=options.application_choice, no_interactive=options.no_interactive,
                      vcs=options.vcs)
        dasboot.bootstrap()

        return 0
    finally:
        if TEMPDIR is not None:
            shutil.rmtree(TEMPDIR)


if __name__ == '__main__':
    sys.exit(main(sys.argv))
```


log

```bash
% python bootstrap.py

Note on Artifact Mode:

Artifact builds download prebuilt C++ components rather than building
them locally. Artifact builds are faster!

Artifact builds are recommended for people working on Firefox or
Firefox for Android frontends, or the GeckoView Java API. They are unsuitable
for those working on C++ code. For more information see:
https://developer.mozilla.org/en-US/docs/Artifact_builds.

Please choose the version of Firefox you want to build:
  1. Firefox for Desktop Artifact Mode
  2. Firefox for Desktop
  3. GeckoView/Firefox for Android Artifact Mode
  4. GeckoView/Firefox for Android
Your choice: 2

Looks like you have Homebrew installed. We will install all required packages via Homebrew.

Your version of Mercurial (4.9.1) is sufficiently modern.
Your version of Python (2.7.11) is new enough.
Could not find a Rust compiler.
Will try to install Rust.
Downloading rustup-init... Ok
Running rustup-init...
info: syncing channel updates for 'stable-x86_64-apple-darwin'
info: latest update on 2019-02-28, rust version 1.33.0 (2aa4c46cf 2019-02-28)
info: downloading component 'rustc'
 78.7 MiB /  78.7 MiB (100 %)  10.4 MiB/s ETA:   0 s
info: downloading component 'rust-std'
 51.5 MiB /  51.5 MiB (100 %)  11.5 MiB/s ETA:   0 s
info: downloading component 'cargo'
info: downloading component 'rust-docs'
info: installing component 'rustc'
info: installing component 'rust-std'
info: installing component 'cargo'
info: installing component 'rust-docs'
info: default toolchain set to 'stable'

  stable installed - rustc 1.33.0 (2aa4c46cf 2019-02-28)


Rust installation complete. You should now have rustc and cargo
in /Users/username/.cargo/bin

The installer tries to add these to your default shell PATH, so
restarting your shell and running this script again may work.
If it doesn't, you'll need to add the new command location
manually.

If restarting doesn't work, edit your shell initialization
script, which may be called ~/.bashrc or ~/.bash_profile or
~/.profile, and add the following line:

    source /Users/username/.cargo/env

Then restart your shell and run the bootstrap script again.


The Firefox build system and related tools store shared, persistent state
in a common directory on the filesystem. On this machine, that directory
is:

  /Users/username/.mozbuild

If you would like to use a different directory, hit CTRL+c and set the
MOZBUILD_STATE_PATH environment variable to the directory you'd like to
use and re-run the bootstrapper.

Would you like to create this directory? (Yn): (Yn): Y
Creating global state directory: /Users/username/.mozbuild

Mozilla recommends a number of changes to Mercurial to enhance your
experience with it.

Would you like to run a configuration wizard to ensure Mercurial is
optimally configured? (Yn): Y
================================================================================
Ensuring https://hg.mozilla.org/hgcustom/version-control-tools is up to date at /Users/username/.mozbuild/version-control-tools
applying clone bundle from https://hg.cdn.mozilla.net/hgcustom/version-control-tools/ff0db1f4639425723040dea6e41609eee695134c.zstd.hg
adding changesets
adding manifests
adding file changes
added 6929 changesets with 16622 changes to 2777 files
finished applying clone bundle
searching for changes
no changes found
6929 local changesets published
1509 files updated, 0 files merged, 0 files removed, 0 files unresolved
(activating bookmark @)
================================================================================
This wizard will guide you through configuring Mercurial for an optimal
experience contributing to Mozilla projects.

The wizard makes no changes without your permission.

To begin, press the enter/return key.

You don't have a username defined in your Mercurial config file. In order
to author commits, you'll need to define a name and e-mail address.

This data will be publicly available when you send commits/patches to others.
If you aren't comfortable giving us your full name, pseudonames are
acceptable.

(Relevant config option: ui.username)
What is your name? username
What is your e-mail address? example@example.example
setting ui.username=username <example@example.example>

Mercurial has implemented some functionality behind ui.tweakdefaults config,
that most users would like by default, but would break some workflows due to
backwards compatibility issues.
You can find more info by running:

  $ hg help config.ui

and checking the "tweakdefaults" section.

Would you like to enable these features (Yn)?  Y
Mercurial is not configured to produce diffs in a more readable format.

Would you like to change this (Yn)?  Y
Mercurial can provide richer terminal interactions for some operations
by using the popular "curses" library.

Would you like to enable "curses" interfaces (Yn)?  Y
Various extensions provide functionality to rewrite repository history. These
enable more powerful - and often more productive - workflows.

If history rewriting is enabled, the following extensions will be enabled:

absorb
   `hg absorb` automatically squashes/folds uncommitted changes in the working
   directory into the appropriate previous changeset. Learn more at
   https://gregoryszorc.com/blog/2018/11/05/absorbing-commit-changes-in-mercurial-4.8/.

histedit
   `hg histedit` allows interactive editing of previous changesets. It presents
   you a list of changesets and allows you to pick actions to perform on each
   changeset. Actions include reordering changesets, dropping changesets,
   folding multiple changesets together, and editing the commit message for
   a changeset.

rebase
   `hg rebase` allows re-parenting changesets from one "branch" of a DAG
   to another. The command is typically used to "move" changesets based on
   an older changeset to be based on the newest changeset.

Would you like to enable these history editing extensions (Yn)?  Y

The evolve extension is a Mercurial extension for faster and
safer mutable history. It implements the changeset evolution concept
for Mercurial, allowing for safe and simple history re-writing. It
includes some new commands such as fold, prune and amend which may
improve your user experience with Mercurial.

The evolve extension is recommended for working with Firefox repositories.
More information about changeset evolution can be found by running:

  $ hg help evolution

as well as:

  $ hg help -e evolve

once the `evolve` extension is enabled.

(Relevant config option: extensions.evolve)

Would you like to enable the evolve extension? (Yn)  Y
adding changesets
adding manifests
adding file changes
added 4121 changesets with 8191 changes to 424 files
new changesets bbeef801409c:7d54a538dd1e
updating to branch stable
277 files updated, 0 files merged, 0 files removed, 0 files unresolved
Evolve was downloaded successfully.
The fsmonitor extension integrates the watchman filesystem watching tool
with Mercurial. Commands like `hg status`, `hg diff`, and `hg commit`
(which need to examine filesystem state) can query watchman to obtain
this state, allowing these commands to complete much quicker.

When installed, the fsmonitor extension will automatically launch a
background watchman daemon for accessed Mercurial repositories. It
should "just work."

Would you like to enable fsmonitor (Yn)?  Y
Enable logging of commands to help diagnose bugs and performance problems (Yn)  Y
The firefoxtree extension makes interacting with the multiple Firefox
repositories easier:

* Aliases for common trees are pre-defined. e.g. `hg pull central`
* Pulling from known Firefox trees will create "remote refs" appearing as
  tags. e.g. pulling from fx-team will produce a "fx-team" tag.
* The `hg fxheads` command will list the heads of all pulled Firefox repos
  for easy reference.
* `hg push` will limit itself to pushing a single head when pushing to
  Firefox repos.
* A pre-push hook will prevent you from pushing multiple heads to known
  Firefox repos. This acts quicker than a server-side hook.

The firefoxtree extension is *strongly* recommended if you:

a) aggregate multiple Firefox repositories into a single local repo
b) perform head/bookmark-based development (as opposed to mq)

(Relevant config option: extensions.firefoxtree)

Would you like to activate firefoxtree (Yn)?  Y
The "clang-format" extension provides execution of clang-format at the commit steps.
It relies on ./mach clang-format directly.
Would you like to activate clang-format (Yn)?  Y
The "format-source" extension provides a way to run code-formatting tools in a way that
avoids conflicts related to this formatting when merging/rebasing code across the
reformatting.
We encourage you to use this extension specially for formatting and managing C/C++
source code. An example of a .hgrc configuration that uses our embedded clang-format
utility from 'mach' is as follows:
[format-source]
clang-format = [Path To Mozilla Repo]/mach clang-format --assume-filename $HG_FILENAME -p
clang-format:configpaths = .clang-format, .clang-format-ignore
clang-format:fileext = .cpp, .c, .h

If `clang-format` is not present under `[format-source]` a default configuration will be used
that is embedded in this extension. The default configuration can be used in most cases.
Would you like to activate format-source (Yn)?  Y
It is common to want a quick view of changesets that are in progress.

The ``hg wip`` command provides such a view.

Example Usage:

  $ hg wip
  @  5887 armenzg tip @ Bug 1313661 - Bump pushlog_client to 0.6.0. r=me
  : o  5885 glob mozreview: Improve the error message when pushing to a submitted/discarded review request (bug 1240725) r?smacleod
  : o  5884 glob hgext: Support line breaks in hgrb error messages (bug 1240725) r?gps
  :/
  o  5883 mars mozreview: add py.test and demonstration tests to mozreview (bug 1312875) r=smacleod
  : o  5881 glob autoland: log mercurial commands to autoland.log (bug 1313300) r?smacleod
  :/
  o  5250 gps ansible/docker-hg-web: set USER variable in httpd process
  |
  ~

(Not shown are the colors that help denote the state each changeset
is in.)

(Relevant config options: alias.wip, revsetalias.wip, templates.wip)

Would you like to install the `hg wip` alias (Yn)?  Y
The ``hg smart-annotate`` command provides experimental support for
viewing the annotate information while skipping certain changesets,
such as code-formatting changes.

Would you like to install the `hg smart-annotate` alias (Yn)?  Y
Will you be submitting commits to Mozilla (Yn)?  Y
Commits to Mozilla projects are typically sent to Phabricator. This is the
preferred code review tool at Mozilla.
Phabricator installation instructions are here
http://moz-conduit.readthedocs.io/en/latest/phabricator-user.html

The push-to-try extension generates a temporary commit with a given
try syntax and pushes it to the try server. The extension is intended
to be used in concert with other tools generating try syntax so that
they can push to try without depending on mq or other workarounds.

(Relevant config option: extensions.push-to-try)

Would you like to activate push-to-try (Yn)?  Y
Your config file needs updating.
Would you like to see a diff of the changes first (Yn)?  Y
--- hgrc.old
+++ hgrc.new
@@ -0,0 +1,39 @@
+[ui]
+username = username <example@example.example>
+tweakdefaults = true
+interface = curses
+[diff]
+git = true
+showfunc = true
+[extensions]
+absorb =
+histedit =
+rebase =
+evolve = /Users/username/.mozbuild/evolve/hgext3rd/evolve
+fsmonitor =
+blackbox =
+firefoxtree = /Users/username/.mozbuild/version-control-tools/hgext/firefoxtree
+clang-format = /Users/username/.mozbuild/version-control-tools/hgext/clang-format
+format-source = /Users/username/.mozbuild/version-control-tools/hgext/format-source
+push-to-try = /Users/username/.mozbuild/version-control-tools/hgext/push-to-try
+[alias]
+wip = log --graph --rev=wip --template=wip
+smart-annotate = annotate -w --skip ignored_changesets
+[revsetalias]
+wip = (parents(not public()) or not public() or . or (head() and branch(default))) and (not obsolete() or orphan()^) and not closed() and not (fxheads() - date(-90))
+ignored_changesets = desc("ignore-this-changeset") or extdata(get_ignored_changesets)
+[templates]
+wip = '{label("wip.branch", if(branches,"{branches} "))}{label(ifeq(graphnode,"x","wip.obsolete","wip.{phase}"),"{rev}:{node|short}")}{label("wip.user", " {author|user}")}{label("wip.tags", if(tags," {tags}"))}{label("wip.tags", if(fxheads," {fxheads}"))}{if(bookmarks," ")}{label("wip.bookmarks", if(bookmarks,bookmarks))}{label(ifcontains(rev, revset("parents()"), "wip.here"), " {desc|firstline}")}'
+[color]
+wip.bookmarks = yellow underline
+wip.branch = yellow
+wip.draft = green
+wip.here = red
+wip.obsolete = none
+wip.public = blue
+wip.tags = yellow
+wip.user = magenta
+[experimental]
+graphshorten = true
+[extdata]
+get_ignored_changesets = shell:cat `hg root`/.hg-annotate-ignore-revs 2> /dev/null || true

Write changes to hgrc file (Yn)?  Y
Your hgrc file is currently readable by others.

Sensitive information such as your Bugzilla credentials could be
stolen if others have access to this file/machine.

Would you like to fix the file permissions (Yn)  Y
Changing permissions of /Users/username/.hgrc

If you would like to clone the mozilla-unified Mercurial repository, please
enter the destination path below.

Destination directory for Mercurial clone (leave empty to not clone): ./
Destination directory './' is not empty.

Would you like to clone to './mozilla-unified' instead?
  1. Yes
  2. No, let me enter another path
  3. No, stop cloning
Your choice: 1
Cloning Firefox Mercurial repository to ./mozilla-unified
pulling from https://hg.mozilla.org/mozilla-unified
applying clone bundle from https://hg.cdn.mozilla.net/mozilla-unified/46a995ea433fa8014a3aaed81cd6e4d81efb7928.zstd-max.hg
adding changesets
adding manifests
adding file changes
transaction abort!
rollback completed
abort: Connection reset by peer


Failed to pull from hg.mozilla.org.

This is most likely because of unstable network connection.
Try running `hg pull https://hg.mozilla.org/mozilla-unified` manually, or
download mercurial bundle and use it:
https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Source_Code/Mercurial/Bundles

Source code can be obtained by running

    hg clone https://hg.mozilla.org/mozilla-unified

Or, if you prefer Git, by following the instruction here to clone from the
Mercurial repository:

    https://github.com/glandium/git-cinnabar/wiki/Mozilla:-A-git-workflow-for-Gecko-development

Or, if you really prefer vanilla flavor Git:

    git clone https://github.com/mozilla/gecko-dev.git


Build system telemetry

Mozilla collects data about local builds in order to make builds faster and
improve developer tooling. To learn more about the data we intend to collect
read here:
https://firefox-source-docs.mozilla.org/build/buildsystem/telemetry.html.

If you have questions, please ask in #build in irc.mozilla.org. If you would
like to opt out of data collection, select (N) at the prompt.

Would you like to enable build system telemetry? (Yn):

Thanks for enabling build telemetry! You can change this setting at any time by editing the config file `/Users/username/.mozbuild/machrc`


Installing Stylo and NodeJS packages requires a checkout of mozilla-central.
Once you have such a checkout, please re-run `./mach bootstrap` from the
checkout directory.
```

## ref
https://developer.mozilla.org/ja/docs/Mozilla/Developer_Guide/Build_Documentation/Mac_OS_X_Build_Prerequisites
