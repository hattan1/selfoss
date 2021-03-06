+++
title = "Using Nix"
weight = 20
+++

While by no means necessary and you can install all dependencies [manually](@/docs/development/setting-up.md), [Nix package manager](https://nixos.org/download.html) can download all dependencies for development and make them available automatically. It can be used on Linux systems in parallel with your system package manager, and even on MacOS (though list of supported packages is narrower there).

After you install Nix, you just need to run `nix-shell` command in the selfoss repository, and you will find yourself in a development environment with all the necessary dependencies on `PATH`.

Or, even nicer, you can install [direnv](https://direnv.net/) and your terminal will load the Nix-based development environment automatically when you `cd` into the selfoss repository directory.

## How does Nix work {#nix}

Nix package manager evaluates package descriptions written in the *Nix* expression language and then turns them into concrete instructions for building packages (*\*.drv* files). Those can then be built or, if the package has already been built by Nixpkgs’ infrastructure, a prebuilt package can be downloaded from binary cache.

Nix is a full-fledged functional programming language (think JSON with functions) and the expressions describing packages are just arbitrary files. Nix community also has a central repository called [*Nixpkgs*](https://github.com/NixOS/nixpkgs) that contains thousands of software packages plus library of functions to make creating package descriptions more convenient.

Nix provides, among other things, `nix-shell` command. Running it will evaluate a dummy package described in `shell.nix` file and place you into a new shell environment with dependencies of that package added to `PATH` environment variable. This will allow you to run them as if you installed them any other way.

Nix language allows loading files containing Nix expressions using `import "path"` command and downloading git repositories and returning paths they were cloned into using `builtins.fetchGit { url = "foo"; rev = "bar"; sha256 = "xxx"; }` function. Using these primitives,[^flakes] we load a specific snapshot (git commit) from Nixpkgs repository and use PHP, composer, npm and other dependencies from there in our shell environment.

[^flakes]: Recently, *Flakes*, an experimental, more high-level method for expressing dependencies between git repositories has been introduced. It allows us to specify dependency on other repositories like Nixpkgs declaratively in `flake.nix` file, and separates the information about pinned versions into a `flake.lock` file instead of the low-level functions. Our `shell.nix` is actually a [compatibility shim](https://github.com/edolstra/flake-compat/) that calls `builtins.fetchGit` function with the data from the lock file.

## Our set-up  {#selfoss-nix}

As we have already mentioned, we describe the development environment in `flake.nix` and we pull some packages from Nixpkgs repository. For maintenance reasons, Nixpkgs usually only contains a single version of each package or, in case of platforms like PHP, single version of each supported branch. Since selfoss aims to support even shared hosts with older PHP versions, we have to build those versions ourselves. Fortunately, it is quite easy using existing Nixpkgs infrastructure – we maintain [a repository](https://github.com/fossar/nix-phps) that contains expressions for those versions.

## Bumping pinned dependencies {#bumping}

The pinned Nixpkgs version can be updated with `nixUnstable` using `nix flake update --recreate-lock-file`, or with stable Nix using `nix-shell -I nixpkgs=channel:nixos-unstable -p nixUnstable --run 'nix --experimental-features "nix-command flakes" flake update --recreate-lock-file'`.

## Optimizing the workflow {#optimizing}

Nix uses binary cache to avoid building packages from Nixpkgs over and over again but the packages we created for old PHP versions are obviously not cached by the official Nixpkgs cache. You can install [Cachix](https://docs.cachix.org/installation.html) and run `cachix use fossar` to enjoy the same benefits for those packages as well. It is not necessary for the default PHP version, though, only if you want to test selfoss on one of the [unmaintained versions](#switching-php).

As mentioned above, we are not actually using Flakes but emulate them using stable Nix features. With `nixUnstable`, we can use Flakes directly and benefit from dramatically increased performance. After you install `nixUnstable` and enable the experimental features `echo 'experimental-features = nix-command flakes' >> ~/.config/nix/nix.conf`, you can use `nix develop` instead of `nix-shell` or include [nix-direnv](https://github.com/nix-community/nix-direnv) into your direnv configuration to make direnv use Flakes as well.

## Switching PHP versions {#switching-php}

By default, the environment will contain the default PHP version from Nixpkgs. You can change the version by replacing `matrix.php = "php";` with `matrix.php = "phpXY";` in `flake.nix`, where `X` and `Y` are major and minor version respectively.

Supported versions depend on which versions are in the pinned version of Nixpkgs in [`pkgs/development/interpreters/php`](https://github.com/NixOS/nixpkgs/tree/nixpkgs-unstable/pkgs/development/interpreters/php) and whichever versions we are keeping in [our repository](https://github.com/fossar/nix-phps).

After you change the value, exit the shell and start a new one, or if you are using `direnv`, execute `touch shell.nix` to trigger the reload of the environment.
