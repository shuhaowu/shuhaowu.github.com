title: Setting up Intellij with CyanogenMod/AOSP development
date: 2014-04-27 23:05
published: yes

Recently, I discovered a way to import the [CyanogenMod][1] source into Intellij.
Since the documentation in this area is severely lacking, I thought I might
share my experience. There are several things that I have yet to figure out,
but the basic setup can be done in Intellij fairly simply (write code, browse
code with CTRL+Click).

[1]: http://www.cyanogenmod.org/

First, let's assume that you have successfully cloned the CyanogenMod/AOSP
repository and built a version of android. The AOSP documentation for IDE
development only [documents Eclipse][2] and it is fairly out of date. Google
included a largely unupdated tool something called `idegen` under `development/tools`.
However, it does generate correct mostly correct IntelliJ configurations.

[2]: http://source.android.com/source/using-eclipse.html

-------------------------------------------------------------------------------

Before getting started, you first need to make sure that your IDE have lots of
RAM allocated to it. You can modify the file idea.vmoptions and idea64.vmoptions
and add these:

    -Xms748m
    -Xmx748m

Next, you need to open up idea.properties and change
`idea.max.intellisense.filesize` to something like 5000 or more. If we don't do
this IntelliJ won't parse some R.java files generated.

*Note: these files are located in the IntelliJ/install/location/bin*

Now go to your android root directory. Let's assume that the android root
directory is `~/cm`.

    $ cd ~/cm

Compile the idegen tools:

    $ cd development/tools/idegen
    $ mm

Go back to the root directory and run the tool:

    $ cd ~/cm
    $ development/tools/idegen/idegen.sh

Go to IntelliJ, and setup an Oracle Java 6 SDK with **no libraries**. That's
right, remove all of the jars from the "Classpath" tab.

[![JDK 1.6 no libraries](/static/img/aosp-intellij/jdk-no-lib.png)][img1]

[img1]: /static/img/aosp-intellij/jdk-no-lib.png

At this point you can open up the android.ipr file generated in the android root
directory with IntelliJ and you should have AOSP imported! However, we're not
done as there is some setup that needs to be fixed:

Go to File -> Project Structure. Remove all of the dependencies
that ends with a .jar. This should leave you with only <Module Source> and the
1.6 sdk with no libraries.

[![JDK 1.6 no libraries](/static/img/aosp-intellij/project-structure-dep.png)][img2]

[img2]: /static/img/aosp-intellij/project-structure-dep.png

Lastly, go to the Sources tab and browse to `out/target/common/R`. Right click
on it and click Source. Apply.

[![JDK 1.6 no libraries](/static/img/aosp-intellij/project-structure-sources.png)][img3]

[img3]: /static/img/aosp-intellij/jdk-no-lib.png

Synchronize the project. Now you're done! Have fun hacking Android with
IntelliJ.

[![JDK 1.6 no libraries](/static/img/aosp-intellij/done.png)][img4]

[img4]: /static/img/aosp-intellij/done.png
