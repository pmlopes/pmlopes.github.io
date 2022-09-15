---
layout: post
title: Secure Vert.x IDE setup
date: 2022-11-15 00:00:00 GMT
---

I recently had to re-install my laptop and noticed that my IDE stop giving me the hints I was used to get regarding
secure code tips.

I'm putting the short setup here so, I can always recall it whenever I have to go over the process again in the future.

First of all I use both a mix of command line and graphical tools. This means that at the CLI level I have installed:

* maven
* openjdk

Usually I just rely on the linux packages from the distribution. While maven usually just works as vert.x builds are not
that fancy, sometimes I need to switch JDKs. To switch JDK I've this simple `bash` script in my `.bashrc`:

```shell
export JVM_PATH=${PATH}
_comp_jvm () {
    # Get list of directories
    # $2 is the word being completed
    COMPREPLY=($(compgen -d "${HOME}"/.jdks/"$2"))
    # Reduce to basenames
    COMPREPLY=("${COMPREPLY[@]##*/}")
}
complete -F _comp_jvm jvm
jvm () {
    # Set the path to use the choosen JVM by default
    export PATH="${HOME}"/.jdks/"$1"/bin:${JVM_PATH}
    # Set JAVA_HOME
    export JAVA_HOME="${HOME}"/.jdks/"$1"
    java -version
}
```

This will allow me to switch from JVM at the CLI cycling any JDKs installed in `~/.jdks`. This is usually the location
`IntelliJ Idea` will download JDKs, so I don't rely on any other tools. The `_comp_jvm` function just allows me to list
the available JDKs using `Tab` completion.

For graphical development I'm using `IntelliJ Idea`. There are a couple of features I enable. Once you open a vert.x
module you can enable `null` checks for vert.x code by:

1. Go to settings
2. Build, Execution, Deployment
3. Compiler
4. Nullable Annotations
5. `+` (add)
6. `io.vertx.codegen.annotations.Nullable`

![IntelliJ Idea](/assets/images/blog/intellij-setup.jpeg)

After this we can add the `SpotBugs` plugin to give you extra hints of potential bugs in your code:

![SpotBugs Plugin](/assets/images/blog/spotbugs.png)

While IDEA will already give you good hints on your code, I'm more interested on a sub-plugin, so you need to:

1. Go to settings
2. Tools
3. SpotBugs
5. `+` (add)
6. `find-sec-bugs`

![SpotBugs Plugin](/assets/images/blog/find-sec-bugs.png)

This will allow us to run extra checks related to security.

That's it for now!

Happy coding!
