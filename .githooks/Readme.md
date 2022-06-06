# Git Hooks

Contains git hooks used with proenv.

## Usage

To use these hooks, client repos map a branch of this repo to their top level `.githooks` hidden directory as either a submodule or a subtree. Rather than map `main`, a new branch should be created with the name of the parent repository using the hooks.

See [git-hooks-example](https://github.com/optdyn/git-hooks-example.git), it maps this git-hooks repository's `git-hooks-example` to its `.githooks` directory. Once the branch is mapped as a sub[module|tree], the client repository can be cloned with:

```shell
curl -fsSL \
  https://raw.githubusercontent.com/optdyn/post-clone/master/bin/clone | \
  bash -s -- http://github.com/optdyn/git-hooks-example.git
```

## Why?

For consistent git configurations and mandating dev policy, cloned repositories should have certain hooks enabled client-side. Things like comment formats, avoiding password leakage, and other checks for security and consistency are a must especially for enterprise development teams. ***Unfortunately, there's no direct mechanism to enforce client-side hooks.***

> **PROBLEMS**: For various reasons, but primarily due to security, client-side git hooks cannot be imposed when cloning repositories. There is no such thing as a `post-clone` hook for these reasons. Without such a facility, setup code cannot be triggered when repositories are cloned nor client side hooks mandated.

Furthermore projects often have additional environment set up requirements beyond cloning the repository. Devs are expected to follow instructions to set up their environment to edit, test, run, debug and deploy the project's assets. Considerate project authors may provide scripts to enable developers to quickly set up environments.

## How?

Unlike `post-commit`, `post-checkout`,.. etc, Git has no `post-clone` hook (for security reasons). Git does however invoke the `post-checkout` hook right after cloning a repo and the [post-clone](https://github.com/git-hook/post-clone) project uses this fact to hack together what almost ***mimics*** a `post-clone` hook.

Why almost? The mechanism **DOES NOT** work directly with direct cloning via `git clone` commands. It only works when used with a script. The script can be curl downloaded and piped to bash like this:

```shell
local -r cloned_repo="http://github.com/optdyn/git-hooks-example.git"
curl -fsSL \
  https://raw.githubusercontent.com/optdyn/post-clone/master/bin/clone | \
  bash -s -- "${cloned_repo}"
```

> All projects using this pattern should have this instruction at the head of their Readme.md files 

The simple `clone` script fetches the ***post-clone*** repository into a temporary location and uses it as the git template for cloning the target `${cloned_repo}` on the client system. This installs the ***post-clone*** repo's contents (`bin` and `hooks`) into the cloned repository's `.git` folder which contains an active `post-checkout` hook.

As part of the hack, `git clone` of the target repo invokes the `post-checkout` script locally in the cloned target repository's `.git/hooks` folder immediately after cloning. The `post-checkout` script deletes itself by deleting the `hooks` folder copied from the temporary ***post-clone*** repo used as the clone template. After deleting the `hooks` directory, it then links the target repo's `${repo}/.githooks` to  `${repo}/.git/hooks`. This essentially registers the target cloned repos hooks.

```shell
# from https://github.com/git-hook/post-clone/blob/master/hooks/post-checkout
cd ${repo}/.git
rm -rf hooks
ln -sf ../.githooks ./hooks
```

The `post-checkout` script supplied by the ***post-clone*** repo in its `hooks` directory then, if present, invokes `${repo}/.githooks/post-clone` or executable scripts contained in a `${repo}/.githooks/post-clone.d` directory in listing order. This mimics `post-clone` hook functionality using this curl downloaded clone script.

> A single script target is allowed for git hooks. This can get problematic so the Debian dot.d directory pattern is used where the main hook simply executes executable files in a directory with the <hook-name>.d name.

I have made a few changes to the original project by Eric Crosson [git-hook/post-clone](https://github.com/git-hook/post-clone):

* Use a hidden `.githooks` folder in target repos instead of `hooks` folders

  > **DONE**: See [optdyn/post-clone: Implementation of a git post-clone hook (github.com)](https://github.com/optdyn/post-clone) 

* Enable multiple scripts per hook in a Debian style hooks directory: i.e. `pre-commit.d`

  > **DONE**: For all hooks with dot.d directories, but with `deleteme` placeholder for now

