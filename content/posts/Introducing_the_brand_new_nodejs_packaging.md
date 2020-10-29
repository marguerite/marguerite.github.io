---
title: "Introducing the brand new nodejs packaging"
date: 2016-01-23T00:00:00+08:00
draft: false
---
There has been no activities for nodejs in openSUSE for a while. (Since 13.2) But it doesn't mean it's dead. It's actually evolving.

Today the brand-new [nodejs-packaging](https://github.com/marguerite/nodejs-packaging) answers all the questions.

For a long time and traditionally we openSUSE prefer to keep consistence with Fedora in RPM packaging (Although small differences still present). So did nodejs packaging. We used the nodejs-packaging tool from Fedora to package for openSUSE before. But it introduced lots of troubles:

The old nodejs-packaging analyzes package.json on module basis. That is, every module is a package, just like things we do for Perl, Python and Ruby. But wait, how many nodejs modules exist? 2 millions. And how many modules that a nodejs application may depend on? For npm, it's 200+; for jshint, it's 200+; for gulp, it's 95. No need to mention the much more complicated ones like atom editor, jangouts, or yakyak.

We used to have 400+ packages in devel:languages:nodejs, while providing only 2 outdated applications (azure and npm). Because it was a nightmare for maintenance. Take npm as example:

It updates almost every week. There's no way for the maintainer (me, sad) to know what are the new dependencies (nodejs-packaging didn't provide such a feature: you only know the next step, but you can't see the whole map due to the tree hierarchy of dependencies); There's no way for the maintainer to know what are the updated dependencies; and there's no way for the maintainer to know what dependencies are removed in advance too.

It means you have to just start packaging the new npm and examine all the growing 200+ dependencies on the fly. But you have just one week time in total.

Even if someone would made an automatic packaging tool (actually no one implemented that although we had some discussions before), there're still two difficulties ahead:

You can't just let the packages lay in devel:languages:nodejs, you should submit it to Factory. Well, 200+ submit requests per week with almost the same nodejs- names are just too hard for reviewing.

nodejs has a mechanism that allows modules to place the dependencies they need to local node_modules directories. The initial aim was to give developers the freedom and possibility not to follow upstream so tight and focus on their own development. But it leads to the fact that most of the modules actually depend on older versions of other modules.

And, the "older versions" may different. that is, module A may requires 1.0.0 version of module C, while module B relies on 2.0.0 version of module C, and the latest version of module C is actually 4.0.0.

So you have to maintain 1.x/2.x/4.x versions of module C. in RPM, a package can only provide one version. then funny things happened: nan-0_8,nan-1_0...while considering such case is too common because nodejs upstream officially supports so, the poor multiver feature of the old nodejs-packaging is not enough.

There must be a change.

In my reimplementation of nodejs-packaging, I used bundled packaging on application basis:

If you want to package jslint for example, you just need to package jslint. No need to worry about all its dependencies (200+). They will be bundled into the main package under node_modules directory.

Someone may worry about the duplicate of bundled dependencies across applications. Actually the waste of disk spaces is so small: the standalone gulp takes 95kb while the bundled one takes 495kb. You'll have to write 200~300kb of specfiles for the dependencies if packaged separately. And 400kb is not a liability for modern hard disks I think.

Bundles can be used as RPM dependency too. that is, gulp can rely on gulp-util package. Both of them are bundles.

Dependencies (Provides/Requires) are automatically handled. eg, inner dependencies for gulp-util will not be applied on gulp.

In the future, I may analyze the <name>.json generated and cherry-pick the most common modules into another "big" package to further reduce the waste of spaces.

#### How to package for nodejs application now?

Firstly, break the application into logically reasonable pieces (bundles). eg, I want to package dependencies for our search page for better deployment:

    gulp
    gulp-concat
    gulp-less
    gulp-rename
    gulp-uglify

In the old times, we have to package hundreds of modules. Now we just need to package those five (actually it's 6, because all those stuff need gulp-util).

Second, osc mkpac gulp-util && cd gulp-util. Then run npkg gulp-util (with nodejs-packaging installed on your machine), it will generate

    gulp-util.json
    gulp-util.license
    gulp-util.source

(gulp-util.json is the file used by packaging macros to bundle stuff correctly; gulp-util.license contains the collected licenses of all things in the bundle; gulp-util.source is the formatted text for Source tags in RPM specfile)

and download all the tarballs of dependencies plus the main gulp-util tarball inside current directory.

Third, `cp -r /usr/share/npkg/template ./gulp-util.spec`. Fill the blanks, osc add the sources and submit.

Everything is okay now. You can consider yourself just packaged 200+ packages in one shoot.
How to use bundles as RPM dependencies?

Just write `Requires: npm(gulp-util) = 3.0.7` in specfile and run `npkg gulp gulp:3.0.7`.

In the foreseeable future, devel:languages:nodejs will be active again. More applications will be added soon. And the nodejs packaging policy on our wiki will be discussed and updated too.

If you run into any abnormal situation eg, extra dependencies are detected (a self-fulfilled bundle should not rely on outside dependencies unless told so), usually it's a bug that should be looked into by me. Feel free to report on github.

PS: The new nodejs-packaging is fully compatible with the old one, you can still package in the old way. Most of the times you won't be aware of the changes. But personally I think the further you walk in the wrong direction, the more
painful it will be.
