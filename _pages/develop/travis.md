---
title: Travis
section: Extend:Development:Tools
---

{% include notice icon="note" content="[SciJava CI has migrated from Travis CI to Github Actions](https://forum.image.sc/t/scijava-ci-migration-from-travis-ci-to-github-actions/57573). See the [GitHub Actions](https://imagej.net/develop/github-actions) page for configuration instructions, including how to migrate existing projects from Travis CI." %}

{% include img src='logos/travis' align='right' width=150 caption='**Travis CI:** Build your code in the cloud!' %}

[Travis CI](https://travis-ci.org/) is a tool for [continuous integration](/develop/project-management#continuous-integration). It has excellent integration with [GitHub](/develop/github), and is very useful for automating builds, deployment and other tasks.

# Services

[SciJava](/libs/scijava) projects use Travis in a variety of ways:

-   Perform builds of SciJava projects. Travis deploys `SNAPSHOT` builds to the [SciJava Maven repository](https://maven.scijava.org/) in response to pushes to each code repository's `master` branch. So any downstream projects depending on a version of `LATEST` for a given component will match the last successful Travis build—i.e., the latest code on `master`.
-   Run each project's associated {% include wikipedia title='Unit testing' text='unit tests'%}. Travis is instrumental in early detection of new bugs introduced to the codebase.
-   Perform [releases](/develop/releasing) of [SciJava](/libs/scijava) projects. Travis deploys release builds to the appropriate Maven repository—typically either the SciJava Maven repository or [OSS Sonatype](https://oss.sonatype.org/).
-   Keep the [javadoc](/develop/source#javadocs) site updated.
-   Keep other web resources updated.

# Automatic Deployment of Maven Artifacts

Deploying your library to a [Maven](/develop/maven) repository makes it available for other developers. It is also a [contribution requirement for the Fiji project](/contribute/fiji).

## Requirements

-   Host your [open-source](/licensing/open-source) project on [GitHub](/develop/github).
-   Log in to [Travis CI](https://travis-ci.com/auth) with your corresponding GitHub account and enable your repository.
-   Contact an ImageJ admin in [Gitter](/discuss/chat#gitter) or [the Image.sc Forum](http://forum.image.sc/) and request that they file a PR which adds Travis support to your repository.

## Instructions

In order to add Travis CI support to a repository, the SciJava credentials are needed: A) for deploying to Maven repositories; and B) in the case of deploying to OSS Sonatype, for GPG signing of artifacts. If you have a copy of these credentials, and admin access to the relevant repository on GitHub, you can use the [travisify.sh script](https://github.com/scijava/scijava-scripts/blob/master/travisify.sh) to perform the needed operations. This script requires the [travis command line client](https://github.com/travis-ci/travis.rb) to be installed, and you may need to run `travis login` to authenticate first. If you need help, please ask [on the Image.sc Forum](https://forum.image.sc/) in the Development category, or in the [scijava-common channel](https://gitter.im/scijava/scijava-common) on Gitter.

## Testing things which cannot run headless

If your tests require a display (i.e.: they do not pass when run [headless](/learn/headless)), you can use [Xvfb](/learn/headless#xvfb) as follows:

    before_script:
      - "export DISPLAY=:99.0"
      - "sh -e /etc/init.d/xvfb start"
      - sleep 3 # give xvfb some time to start

Of course, you should do this only as a last resort, since the best unit tests should not require a display in the first place.


