# Setting up a Haskell development environment with Nix

- [Introducing Nix](#introducing-nix)
    - [nix-channel](#nix-channel)
    - [nix search](#nix-search)
    - [First using nix-env](#first-using-nix-env)
    - [Alternatively, using nix-shell](#alternatively-using-nix-shell)
- [A simple Haskell environment](#a-simple-haskell-environment)
- [Introducing Direnv and Lorri](#introducing-direnv-and-lorri)
    - [Direnv](#direnv)
    - [Lorri](#lorri)
- [Bonus: Setting up Ghcide to work on our project](#bonus-setting-up-ghcide-to-work-on-our-project)

We'll explore how to quickly setup a reliable environment to hack on our Haskell projects using the **Nix** language and its **Nixpkgs** infrastructure.
As we proceed, I'll introduce some auxiliary tools we can add to our stack to streamline our workflow even further. Let's dive in!

## Introducing Nix

As stated on the project's website, **\"Nix is a powerful package manager for Linux and other Unix systems that makes package management reliable and reproducible\"**.
What's of interest to us here is that Nix is a tool which allows you to setup self-contained environments for all your development needs.
Think of it as a more lightweight and less cumbersome alternative to Docker.

Ok now let's first install it and then I'll explain how it works.

The installation procedure is pretty straightforward, just paste the following in your terminal and you'll be all setup.
~~~bash
curl -L https://nixos.org/nix/install | sh
~~~
You may need to reload your profile to get access to the nix commands.

I'll now introduce the commands we'll use for the matter at hand here.

### nix-channel
Using this command should print something like the following in your terminal.
~~~bash
[romain@clevo-N141ZU:~]$ nix-channel --list
nixpkgs https://nixos.org/channels/nixpkgs-unstable
~~~
What this means is that your nix installation is currently subscribed to the **nixpkgs-unstable** channel.
A channel is tied to a git branch of the [Nixpkgs GitHub repository](https://github.com/NixOS/nixpkgs).
Conceptually, a channel is a set of package as they are defined by the GitHub repository as of the commit it points to, which makes exploring the source for a given package convenient once you've understood how to navigate the GitHub repository.
For example you can see what's the state of the **nixpkgs-unstable** channel by browsing the repository at this url [https://github.com/NixOS/nixpkgs/tree/nixpkgs-unstable](https://github.com/NixOS/nixpkgs/tree/nixpkgs-unstable).\
Beware though, your local nix installation isn't automatically synced with GitHub, you should use the command **nix-channel --update** to bring it up to date, somewhat akin to what **apt update** would do on Debian based distributions.

### nix search

Now that you understand the purpose of nix channels, you'll surely want to search for available packages in a convenient manner.
That's what the **nix search** command brings to the table. Let's say you'd like to install GHC, the **Glasgow Haskell Compiler**, on your machine.\
To search for the package, you'd type the following in your terminal:

~~~bash
[romain@clevo-N141ZU:~]$ nix search ghc
warning: using cached results; pass '-u' to update the cache
* nixpkgs.ghc (ghc)
  The Glasgow Haskell Compiler
* nixpkgs.ghcid (ghcid)
  GHCi based bare bones IDE
* nixpkgs.vimPlugins.ghc-mod-vim (vimplugin-ghcmod-vim)
* nixpkgs.vimPlugins.ghcid (vimplugin-ghcid)
* nixpkgs.vimPlugins.ghcmod (vimplugin-ghcmod-vim)
* nixpkgs.vimPlugins.ghcmod-vim (vimplugin-ghcmod-vim)
* nixpkgs.vimPlugins.neco-ghc (vimplugin-neco-ghc)
* nixpkgs.vimPlugins.necoGhc (vimplugin-neco-ghc)
~~~
As expected, the first result is the one of interest to us.

I'll now demonstrate two ways in which we can get to play with this package.

### First using nix-env

You can install GHC using the **nix-env -i** command. See below:
~~~bash
[romain@clevo-N141ZU:~]$ nix-env -i ghc
installing 'ghc-8.8.3'
*** Elided output for brievity ***
building '/nix/store/b4p2sf432qyfr91ixz9xnl9hw6hqvg9p-user-environment.drv'...
created 52 symlinks in user environment
~~~

And sure enough we now have access to the GHC binary:
~~~bash
[romain@clevo-N141ZU:~]$ whereis ghc
ghc: /nix/store/f6j4lqvx3zmm05wqmpi3867srkghd4vv-user-environment/bin/ghc

[romain@clevo-N141ZU:~]$ ghc --version
The Glorious Glasgow Haskell Compilation System, version 8.8.3
~~~

Now you may wonder what's with the ugly paths given by the **whereis** command. It's time to take a break to explain how nix works under the hood.
Nix uses a central store located at **/nix/store** where you'll find every package in use by your current installation.
The files you'll find there are named like the following **0ywc4dlk8159rj7q5fhkdvm2xrjkfxy3-ghc-8.8.3** where **0ywc4dlk8159rj7q5fhkdvm2xrjkfxy3** is a hash representing every input involved in the build of that version of the package.\
You can then have multiple versions of the same package (even of the same version of the package depending on its input) installed on your machine at the same time.
Those packages aren't in scope by default. What makes those available to you is the user profile Nix sets up with a clever usage of symbolic links.

For example, on my machine, we can see that the **.nix-profile** file present in my home directory is a symbolic link pointing to some directory in **/nix/var**.
Let's follow the trail to see where this leads us.
~~~bash
[romain@clevo-N141ZU:~]$ readlink .nix-profile
/nix/var/nix/profiles/per-user/romain/profile
~~~
The **.nix-profile** entry points to another symlink in the nix profiles directory.
~~~bash
[romain@clevo-N141ZU:~]$ readlink /nix/var/nix/profiles/per-user/romain/profile
profile-71-link
~~~
Which itself points to **profile-71-link**, aptly named because it's the 71th version of my user environment. A new version being built whenever you install, update or remove a package.
~~~bash
[romain@clevo-N141ZU:~]$ readlink /nix/var/nix/profiles/per-user/romain/profile-71-link
/nix/store/f6j4lqvx3zmm05wqmpi3867srkghd4vv-user-environment
~~~
This then points to our current enviroment in the store.
~~~bash
[romain@clevo-N141ZU:~]$ ll /nix/store/f6j4lqvx3zmm05wqmpi3867srkghd4vv-user-environment
total 20
dr-xr-xr-x 2 root root 4096  1 janv.  1970 bin
lrwxrwxrwx 1 root root   57  1 janv.  1970 lib -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/lib
lrwxrwxrwx 1 root root   66  1 janv.  1970 libexec -> /nix/store/yv86xmr68ssdq2ba0yy4zd05qxpvbcf1-yarn2nix-1.0.0/libexec
lrwxrwxrwx 1 root root   60  1 janv.  1970 manifest.nix -> /nix/store/lmsf8qj1zr4fi0aylk6jh9qv4q84x81i-env-manifest.nix
dr-xr-xr-x 6 root root 4096  1 janv.  1970 share
lrwxrwxrwx 1 root root   67  1 janv.  1970 tarballs -> /nix/store/yv86xmr68ssdq2ba0yy4zd05qxpvbcf1-yarn2nix-1.0.0/tarballs
~~~
And, at last, we get to see where the GHC binary is accessed from.
~~~bash
[romain@clevo-N141ZU:~]$ ll /nix/store/f6j4lqvx3zmm05wqmpi3867srkghd4vv-user-environment/bin/
total 116
lrwxrwxrwx 1 root root 61  1 janv.  1970 ghc -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghc
lrwxrwxrwx 1 root root 67  1 janv.  1970 ghc-8.8.3 -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghc-8.8.3
lrwxrwxrwx 1 root root 62  1 janv.  1970 ghci -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghci
lrwxrwxrwx 1 root root 68  1 janv.  1970 ghci-8.8.3 -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghci-8.8.3
lrwxrwxrwx 1 root root 65  1 janv.  1970 ghc-pkg -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghc-pkg
lrwxrwxrwx 1 root root 71  1 janv.  1970 ghc-pkg-8.8.3 -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/ghc-pkg-8.8.3
lrwxrwxrwx 1 root root 65  1 janv.  1970 haddock -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/haddock
lrwxrwxrwx 1 root root 75  1 janv.  1970 haddock-ghc-8.8.3 -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/haddock-ghc-8.8.3
lrwxrwxrwx 1 root root 63  1 janv.  1970 hp2ps -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/hp2ps
lrwxrwxrwx 1 root root 61  1 janv.  1970 hpc -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/hpc
lrwxrwxrwx 1 root root 64  1 janv.  1970 hsc2hs -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/hsc2hs
lrwxrwxrwx 1 root root 76  1 janv.  1970 markdown -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/markdown
lrwxrwxrwx 1 root root 80  1 janv.  1970 markdown.bat -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/markdown.bat
lrwxrwxrwx 1 root root 71  1 janv.  1970 mmd -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd
lrwxrwxrwx 1 root root 75  1 janv.  1970 mmd2all -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2all
lrwxrwxrwx 1 root root 75  1 janv.  1970 mmd2odf -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2odf
lrwxrwxrwx 1 root root 76  1 janv.  1970 mmd2opml -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2opml
lrwxrwxrwx 1 root root 75  1 janv.  1970 mmd2pdf -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2pdf
lrwxrwxrwx 1 root root 75  1 janv.  1970 mmd2rtf -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2rtf
lrwxrwxrwx 1 root root 75  1 janv.  1970 mmd2tex -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/mmd2tex
lrwxrwxrwx 1 root root 81  1 janv.  1970 multimarkdown -> /nix/store/fngkm633j7ga5hg93amzbq1h058z4mxf-multimarkdown-4.7.1/bin/multimarkdown
lrwxrwxrwx 1 root root 74  1 janv.  1970 ob -> /nix/store/gd99xxp41i5dllhpx341rxscgbyzlpqn-obelisk-command-0.8.0.0/bin/ob
lrwxrwxrwx 1 root root 69  1 janv.  1970 patchelf -> /nix/store/58vavyggrv0s48l5fshif4c8vswlhp5x-patchelf-0.9/bin/patchelf
lrwxrwxrwx 1 root root 64  1 janv.  1970 runghc -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/runghc
lrwxrwxrwx 1 root root 70  1 janv.  1970 runghc-8.8.3 -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/runghc-8.8.3
lrwxrwxrwx 1 root root 68  1 janv.  1970 runhaskell -> /nix/store/yzndcg3yq2iaps4zjp8wc4fwwpi9b1pj-ghc-8.8.3/bin/runhaskell
lrwxrwxrwx 1 root root 61  1 janv.  1970 xev -> /nix/store/lzjswkiibs9yr6b47ir6h2vgbrnzp9sv-xev-1.2.3/bin/xev
lrwxrwxrwx 1 root root 71  1 janv.  1970 yarn2nix -> /nix/store/yv86xmr68ssdq2ba0yy4zd05qxpvbcf1-yarn2nix-1.0.0/bin/yarn2nix
lrwxrwxrwx 1 root root 64  1 janv.  1970 zola -> /nix/store/2m7llhrfgjqk064fz7pqhdcaz9x1rb59-zola-0.10.1/bin/zola
~~~

The other binaries you see here are packages I've installed in my environment using the **nix-env -i** method.
You can list them using **nix-env -q**.

~~~bash
[romain@clevo-N141ZU:~]$ nix-env -q
ghc-8.8.3
multimarkdown-4.7.1
obelisk-command-0.8.0.0
patchelf-0.9
xev-1.2.3
yarn2nix-1.0.0
zola-0.10.1
~~~

To uninstall a package, use **nix-env -e**, we'll do this in order to demonstrate the second way of getting GHC in scope.
~~~bash
[romain@clevo-N141ZU:~]$ nix-env -e ghc
uninstalling 'ghc-8.8.3'
~~~

As a side note, if I was to install GHC again right now using **nix-env -i ghc**, it would be nearly instant as all the previously downloaded files are still present in the store.
Nix just has to rebuild the user environment and set up the symlinks accordingly.

Whenever you want to clean the nix store to free up some space on your machine, you can run **nix-collect-garbage -d** which will remove the files in the store which are not part of what's called a **garbage collection root**. Essentially, a **garbage collection root** is some symlink which points to packages in the nix store and which prevents them from being garbage collected.

### Alternatively, using nix-shell

The Nix shell is using the same mechanics but is more intended to setup project environments or to test packages without installing them. This is what we'll use for our Haskell projects later.

First a simple example of getting GHC into scope using **nix-shell -p** which essentially means ***get me into a shell with the following packages in scope***.
~~~bash
[romain@clevo-N141ZU:~]$ ghc
The program ‘ghc’ is currently not installed. You can install it by typing:
  nix-env -iA nixos.ghc

[romain@clevo-N141ZU:~]$ nix-shell -p ghc

[nix-shell:~]$ ghc
ghc: no input files
Usage: For basic information, try the `--help' option.

[nix-shell:~]$ exit

[romain@clevo-N141ZU:~]$ ghc
The program ‘ghc’ is currently not installed. You can install it by typing:
  nix-env -iA nixos.ghc
~~~
Simple enough.
Now what's of interest is that we can provide the packages we want to build a shell for using a nix file. That's what is commonly used in project development.
When invoking **nix-shell**, Nix will look for a **shell.nix** or a **default.nix** file in this order.

The canonical example of this as used in the community is building a shell for the **gnu hello** package, so let's do this.
~~~bash
[romain@clevo-N141ZU:~]$ mkdir nix-shell-example && cd nix-shell-example
~~~
We'll create a **shell.nix** file with the following content.
~~~nix
let
  pkgs = import <nixpkgs> {};
in
pkgs.mkShell {
  buildInputs = [
    pkgs.hello
  ];
}
~~~
Let's explain what we have here.\
First we import **\<nixpkgs\>**, what this means is that we'll have access to our currently subscribed nix-channel set of packages, here **nixpkgs-unstable**, through the **pkgs** variable.
Then we invoke **mkShell** which will build a shell environment for the packages given in **buildInputs**.
That's the same as **nix-shell -p hello** but using declarative nix file.
Here **pkgs.hello** is the same as you'd have found using **nix search hello**.\
Testing it with nix-shell gives us the expected result.
~~~bash
[nix-shell:~/nix-shell-example]$ ll
total 4
-rw-r--r-- 1 romain users 92 juin   2 18:28 shell.nix

[romain@clevo-N141ZU:~/nix-shell-example]$ nix-shell

[nix-shell:~/nix-shell-example]$ hello
Bonjour, le monde !
~~~
There's one problem with our current setup though, that's the reliance on **\<nixpkgs\>**, this isn't what we'd call a truly reproducible environment as it currently depends on the version of the nix channel currently in use on our machine. We'll see how to fix this with the Haskell example.

## A simple Haskell environment

Ok, now that you should have a basic grasp of how Nix works, we'll see how to use it for Haskell development.

We'll proceed with the following simple setup.

~~~bash
[romain@clevo-N141ZU:~/Code/haskell-nix]$ tree .
.
├── default.nix
├── .gitignore
├── haskell-nix.cabal
├── shell.nix
└── src
    └── Main.hs

1 directory, 4 files
~~~

**haskell-nix.cabal**
~~~
cabal-version: >= 1.10
name: haskell-nix
version: 0.1.0
build-type: Simple

executable haskell-nix
  hs-source-dirs: src
  main-is: Main.hs
  default-language: Haskell2010
  build-depends: base
~~~
**Main.hs**
~~~haskell
module Main where

main :: IO ()
main = print "hello, world!"
~~~
Nothing too fancy here.

**shell.nix**
~~~nix
(import ./default.nix).shell
~~~
Now our shell.nix is merely here to access the shell attribute exposed by the default.nix which we'll now focus on.

**default.nix**
~~~nix
let

  nixpkgsRev = "0f5ce2fac0c7";
  compilerVersion = "ghc865";
  compilerSet = pkgs.haskell.packages."${compilerVersion}";

  githubTarball = owner: repo: rev:
    builtins.fetchTarball { url = "https://github.com/${owner}/${repo}/archive/${rev}.tar.gz"; };

  pkgs = import (githubTarball "NixOS" "nixpkgs" nixpkgsRev) { inherit config; };
  gitIgnore = pkgs.nix-gitignore.gitignoreSourcePure;
  
  config = {
    packageOverrides = super: let self = super.pkgs; in rec {
      haskell = super.haskell // {
        packageOverrides = self: super: {
          haskell-nix = super.callCabal2nix "haskell-nix" (gitIgnore [./.gitignore] ./.) {};
        };
      };
    };
  };
  
in {
  inherit pkgs;
  shell = compilerSet.shellFor {
    packages = p: [p.haskell-nix];
    buildInputs = with pkgs; [
      compilerSet.cabal-install
    ];
  };
}

~~~
Now here's the meat of our example. Lets break it apart.

~~~nix
nixpkgsRev = "0f5ce2fac0c7";
compilerVersion = "ghc865";
compilerSet = pkgs.haskell.packages."${compilerVersion}";
~~~
We first introduce 3 completely arbitrary bindings.
- **nixpkgsRev** here is bound to a specific commit of the the GitHub **nixpkgs** repository. This is how we achieve true reproducibility, by not depending on any nix channel and its current state.
- **compilerVersion** is the version of ghc we want to use to build our package.
- **compilerSet** is an alias we set to make our life easier in the remaining of the file. It points to the ghc version specific (ghc865 here) set of haskell packages in the nixpkgs repository.
~~~nix
githubTarball = owner: repo: rev:
  builtins.fetchTarball { url = "https://github.com/${owner}/${repo}/archive/${rev}.tar.gz"; };
~~~
We then showcase how you can define functions in Nix as you would in your favorite language. Here, **githubTarball** is a function which, given a github owner, a github repository and a commit revision will fetch the tarball of the code from GitHub.

~~~nix
pkgs = import (githubTarball "NixOS" "nixpkgs" nixpkgsRev) { inherit config; };
gitIgnore = pkgs.nix-gitignore.gitignoreSourcePure;

config = {
  packageOverrides = super: let self = super.pkgs; in rec {
    haskell = super.haskell // {
      packageOverrides = self: super: {
        haskell-nix = super.callCabal2nix "haskell-nix" (gitIgnore [./.gitignore] ./.) {};
      };
    };
  };
};
~~~
Then you can see what differs from our previous shell example. Here we don't rely on the "magic" **\<nixpkgs\>** but we are specific with which nixpkgs commit we want to build our environment from. Reproducibility again.

You should also notice that we provide a specific **config** to our instantiation of pkgs. This is how we can override parts of it. Here we'll use it to add our **haskell-nix** package to the set of all haskell packages, as if it was initially a part of the package set.\
We also pull in the **gitignoreSourcePure** from the **nix-gitignore** package. This will allow us to tell Nix that the files indicated into our **.gitignore** should be excluded when building our package.

Then we have the **config** binding, that's the same config that is passed to the import of nixpkgs a few lines above.
The **packageOverrides** definition is a bit hairy here but that's the way to override the set of haskell packages of every version of ghc available.\
To introduce our **haskell-nix** package to the set, we rely on **callCabal2nix** which is a utility from the haskell nixpkgs infrastructure. It automatically translates our cabal file into a Nix derivation behind the scenes so that you don't have to worry about this.\
In its most simple form demonstrated here, you only have to specify the name of your package as defined in your cabal file and the path to where your cabal file is located. 
Here we wrap the path given with the **gitIgnore** which takes a list of **.gitignore** files as first argument.

~~~nix
in {
  inherit pkgs;
  shell = compilerSet.shellFor {
    packages = p: [p.haskell-nix];
    buildInputs = with pkgs; [
      compilerSet.cabal-install
    ];
  };
}
~~~
Now we get to the attributes exposed by our **default.nix** file, there are two of them:
- **pkgs**, here **inherit pkgs;** is just another way of writing **pkgs = pkgs;** \. The key **pkgs** is what gets exposed, the value **pkgs** corresponds to the one defined in the **let** part of our file, basically the whole nixpkgs as defined from the commit **0f5ce2fac0c7** modulo addition of our **haskell-nix** package.
- **shell** is what we'll use for our development purpose. The nixpkgs haskell infrastructure provides a **shellFor** utility which is essentially a **mkShell** that understands that it's actually used in a haskell context. You must provide it with an attribute set specifying the haskell packages for which you want to build a development environment for.
To build our shell, we give it our freshly included **haskell-nix** package. The **p** here is a reference to the package set of our chosen ghc version.
You can then use the **buildInputs** attribute to specify everything you need for your development shell. This can include any package of nixpkgs, not only haskell ones. For example, you could include **yarn** here if you're working on some javascript package. In our case, we'll include **cabal-install** since we need to use some of its facilities to hack on our project.

Let's check if this works.

~~~bash
[romain@clevo-N141ZU:~/Code/haskell-nix]$ nix-shell
building '/nix/store/w0z0lil4lmhiiab2iiywi97v4j9gv5cm-cabal2nix-haskell-nix.drv'...
installing

[nix-shell:~/Code/haskell-nix]$ cabal run haskell-nix
Warning: The package list for 'hackage.haskell.org' is 245 days old.
Run 'cabal update' to get the latest list of available packages.
Resolving dependencies...
Build profile: -w ghc-8.6.5 -O1
In order, the following will be built (use -v for more details):
 - haskell-nix-0.1.0 (exe:haskell-nix) (first run)
Configuring executable 'haskell-nix' for haskell-nix-0.1.0..
Preprocessing executable 'haskell-nix' for haskell-nix-0.1.0..
Building executable 'haskell-nix' for haskell-nix-0.1.0..
[1 of 1] Compiling Main             ( src/Main.hs, /home/romain/Code/haskell-nix/dist-newstyle/build/x86_64-linux/ghc-8.6.5/haskell-nix-0.1.0/x/haskell-nix/build/haskell-nix/haskell-nix-tmp/Main.o )
Linking /home/romain/Code/haskell-nix/dist-newstyle/build/x86_64-linux/ghc-8.6.5/haskell-nix-0.1.0/x/haskell-nix/build/haskell-nix/haskell-nix ...
"hello, world!"
~~~

And sure enough this works.
As you can see, cabal is warning me that my package list is greatly out of date here.
You should not pay attention to this as Nix resolves packages through its own machinery. Here, we only use cabal for its building capabilities and not for package management.

To build our package with nix, we'll introduce the **nix-build** command. If you don't pass it a **-Q** argument, it looks for a **default.nix** file in the current directory.
We want to build our **haskell-nix** package which is now exposed through the overridden nixpkgs set so we'll point **nix-build** to it (you don't have to be in the shell to use **nix-build**).

~~~bash
[romain@clevo-N141ZU:~/Code/haskell-nix]$ nix-build -A pkgs.haskell.packages.ghc865.haskell-nix
these derivations will be built:
  /nix/store/kpasmmscjr2hzqrj5zizkkgdrn3r6jj1-haskell-nix-0.1.0.drv
building '/nix/store/kpasmmscjr2hzqrj5zizkkgdrn3r6jj1-haskell-nix-0.1.0.drv'...
setupCompilerEnvironmentPhase
Build with /nix/store/89ln27rjz9xisxcfvvcbm43myd92y280-ghc-8.6.5.
unpacking sources
unpacking source archive /nix/store/a6wsbh4pp0nvhq2bribmm6p0yam0800a-haskell-nix
source root is haskell-nix
*** Elided output for brievity ***
/nix/store/b9h0kd4907n790lig06cdvaawbkr0vl1-haskell-nix-0.1.0
~~~
The last line of output is the path in the nix store where our **haskell-nix** lives.
We now also have a **result** in the current directory which points to it which allows us to conveniently test it.

~~~bash
[romain@clevo-N141ZU:~/Code/haskell-nix]$ ./result/bin/haskell-nix
"hello, world!"
~~~
That's all there is to it.
We can now look how to further improve our setup.
Right now, if you add a dependency in your cabal file, you'll have to exit the shell and reenter it. If you don't do this, cabal will try to fetch the new dependencies itself and bad things will happen so don't.

## Introducing Direnv and Lorri

Those two will bring some nice quality of life improvements, especially when trying to integrate the various haskell IDEs in our development workflow.

### Direnv
As stated on its website, **\"direnv is an extension for your shell. It augments existing shells with a new feature that can load and unload environment variables depending on the current directory\"**. To install direnv you can just use your new favorite package manager and run
~~~bash
nix-env -i direnv
~~~
You then have to hook direnv into your shell. Instructions differ depending on your shell so i'll point you to [Direnv docs](https://direnv.net/docs/hook.html).
To use it with our project, we need to add a **.envrc** at the root of our directory. Everytime you navigate to a directory, direnv will check if there's a **.envrc** file in it and proceed with building the environment it defines.
Let's create this file and use the nix integration.
~~~
-- .envrc
use_nix
~~~
For this to work, you first have to run **direnv allow** in the directory (and again anytime the **.envrc** is modified), that's a security measure to prevent arbitrary scripts from being ran without you first allowing them.
That's it, direnv will now leverage nix-shell as you'd have done manually before to setup the environment.
You don't have to enter the nix-shell explicitly anymore but you still have to run **direnv reload** anytime you modify your cabal file to let Nix bring the new dependency in scope.
That's still a bit cumbersome so let's pull in...

### Lorri

As stated on its [GitHub repository](https://github.com/target/lorri), **\"lorri is a nix-shell replacement for project development. lorri is based around fast direnv integration for robust CLI and editor integration\"**. To install it, simply run
~~~bash
nix-env -i lorri
~~~
Then replace the content of your **.envrc** file with the following
~~~
-- .envrc
eval "$(lorri direnv)"
~~~
For lorri to work, you have to start its daemon by invoking **lorri daemon** in a shell. You don't have to do it in the project directory as it's a system wide process. See [here](https://github.com/target/lorri/blob/master/contrib/daemon.md) if you want to start it automatically as a systemd service.
Lorri will now continuously watch for changes in files referenced by your **shell.nix** file and reload the environment accordingly. A nice benefit of using lorri is that it will create garbage collection roots in your user directory **\~/.cache/lorri/gc\_roots**. What this means is that you won't lose all your project state when running **nix-collect-garbage** . This will essentially prevent you from having to download a massive amount of dependencies again when you get back to working on your project. On the flip side you have to purge those garbage roots from times to times if you don't want your machine to get cluttered.


## Bonus: Setting up Ghcide to work on our project

[Ghcide](https://github.com/digital-asset/ghcide) is, along [haskell-ide-engine](https://github.com/haskell/haskell-ide-engine), one of the current possibilites to get a decent ide environment to hack on haskell code. Those will eventually be merged together into [haskell-language-server](https://github.com/haskell/haskell-language-server) in the near future. We'll use [ghcide-nix](https://github.com/cachix/ghcide-nix) so we'll follow the instructions there.
First, let's install **cachix**, this will allow us to easily add binary caches to our nix installation. A binary cache is a remote server you can fetch precompiled binaries from. Nix will look them up using the hash that you see in the file names located in the nix store. This will essentially prevent you from having to compile the whole universe on your machine. By default, nix will use the nixos binary cache. Let's proceed.
~~~bash
$ nix-env -iA cachix -f https://cachix.org/api/v1/install
$ cachix use ghcide-nix
~~~
That's it, Nix will now look for precompiled binaries in the ghcide-nix cache too, this will come handy with what we are about to do.

Alright, let's just modify our **default.nix** now.

In the let part of our file, we'll add the following
~~~nix
ghcide = (import (githubTarball "cachix" "ghcide-nix" "master") {})."ghcide-${compilerVersion}";
~~~

And in the **buildInputs** of the shell, we'll add ghcide

~~~nix
shell = compilerSet.shellFor {
  packages = p: [p.haskell-nix];
  buildInputs = with pkgs; [
    compilerSet.cabal-install
    ghcide
  ];
};
~~~

That's all, you now need to let nix reload your environment. This could take a while depending on the speed of your bandwith.

Let's run it just to check everything's still ok.

~~~bash
[nix-shell:~/Code/haskell-nix]$ ghcide
ghcide version: 0.1.0 (GHC: 8.6.5) (PATH: /nix/store/8cn9219qr3iq479qvcrn6wh5qr8x635l-ghcide-exe-ghcide-0.1.0/bin/ghcide)
Ghcide setup tester in /home/romain/Code/haskell-nix.
Report bugs at https://github.com/digital-asset/ghcide/issues

Step 1/6: Finding files to test in /home/romain/Code/haskell-nix
Found 1 files

Step 2/6: Looking for hie.yaml files that control setup
Found 1 cradle

Step 3/6, Cradle 1/1: Implicit cradle for /home/romain/Code/haskell-nix
Cradle {cradleRootDir = "/home/romain/Code/haskell-nix", cradleOptsProg = CradleAction: Default}

Step 4/6, Cradle 1/1: Loading GHC Session
Interface files cache dir: /home/romain/.cache/ghcide/da39a3ee5e6b4b0d3255bfef95601890afd80709

Step 5/6: Initializing the IDE

Step 6/6: Type checking the files

Completed (1 file worked, 0 files failed)
~~~

You'll now have to setup the client part of the language server. This will depend of your ide of choice. I personally use [lsp-haskell](https://github.com/emacs-lsp/lsp-haskell) with Emacs which works fine and is not too hard to setup.

Right, that'll conclude our introduction to using nix and haskell together. If you have any questions, feel free to open an issue.

