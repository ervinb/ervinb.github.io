---
layout: article
title: Checkout a specific git commit in a Jenkins job
tags: jenkins
key: jenkins-commit
---

As a good number of Jenkins related articles, this one is also fueled by past frustration. Trying to do the simplest
of things often sets you off on a wild ride of trial and error (mostly error). At which point your Google results are painted purple.
One such thing is checking out a specific git commit during a Jenkins job. On the surface it's a trivial thing to do, and
it is if you don't wade of too far from the road.

Picture this: you want a Jenkins job which is set to run the `master` branch and you can specify any commit from the repository as a parameter,
in which case it will check that commit out and carry on. Simple enough.

## Setup

There is a `commit_sha` parameter which we will ignore if it's empty but check it out otherwise:

![param]({{ site.url }}/assets/images/jenkins-commit/jenkins-job-params.png)

The job is configured to use the Git plugin and the branch specifier is set to `*/master`:

![pipeline]({{ site.url }}/assets/images/jenkins-commit/jenkins-initial-pipeline.png)

Important: `Lightweight checkout` is **not** enabled.

(the repository name is a coincidence)

The `Jenkinsfile` (don't copy this, the correct version is at the bottom):

```groovy
node {
  stage("Hello") {
    print "Hi!"

    if (params.commit_sha?.trim()) {
      print "Checking out commit ${commit_sha}"

      scmVars = checkout([
        $class: 'GitSCM',
        branches: [[name: params.commit_sha]],
      ])
    } else {
      print "Checking out master"

      scmVars = checkout(scm)
    }
  }
}
```

Lastly, we work with two branches:
- `master`
```
8c9963d (HEAD -> master) Commit in 'master'
ce3ed60 Another commit in master
```
- `feat-1`
```
9628c6f (HEAD -> feat-1) Commit in 'feat-1'
```

## Verifying the setup

Running the job without the `commit_sha` parameter works as expected:

![master-init]({{ site.url }}/assets/images/jenkins-commit/master-init.png)

Running the job with `ce3ed60` as the commit parameter also works as expected:

![commit-init]({{ site.url }}/assets/images/jenkins-commit/commit-init.png)

Now let's build the commit from a branch __other than master__, commit `9628c6f` from `feat-1`.

![commit-fails]({{ site.url }}/assets/images/jenkins-commit/commit-there-but-fails.png)

```
ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.
```

The commit is obviously present but it still fails to check it out. What's happening here?
Ah right, the job is limited to the `master` branch only, so you remove that:

![any-branch]({{ site.url }}/assets/images/jenkins-commit/any-branch.png)

Start the job with the commit, and:

```
ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.
```

What?!

## Solution

The issue is twofold:
  1. The initial checkout (configured in the UI, not in the Jenkinsfile) is partial (sparse)
  2. The second checkout, defined in the `if/else` block, is working on a local copy of the repo which is non-existent

The first checkout, visible in the log, is defined in the Pipeline setup, and it's concerned with checking out the Jenkinsfile only
and __only__ that. If you list the contents of the current directory before our own `checkout` call, you will find an empty directory.

Actually, the second run where we used the previous commit from `master` worked because the repository was fully checked out in the
initial job, when `commit_sha` wasn't defined and `checkout(scm)` ran. This made the state 'dirty' and the subsequent job
knew about that commit.

The `checkout` in the `if` block, which tries to use the revision from the parameter, doesn't have a repo to work with.
Intuitively, you'd expect it to know about the repository configured in the settings, and it actually does in the specific
case of `checkout(scm)`.
The `scm` variable comes from the job's settings and [equals to](https://support.cloudbees.com/hc/en-us/articles/226122247-How-to-customize-Checkout-for-Pipeline-Multibranch) something like this:

```
checkout([
  $class: 'GitSCM',
  branches: scm.branches,
  extensions: scm.extensions,
  userRemoteConfigs: scm.userRemoteConfigs
])
```

But in our setup we have:
```
checkout([
  $class: 'GitSCM',
  branches: [[name: params.commit_sha]]
])
```

The key takeaway here is that `userRemoteConfigs` __doesn't__ default to `scm.userRemoteConfigs`. When that field is omitted, the checkout step
works on what it has locally, which is nothing in this case, as the initial checkout is sparse. The error message does make sense after all.

The `else` block, doing the `checkout(scm)`, worked because it does set that field based on `scm` (as seen above).

So the fix is:
```
checkout([
  $class: 'GitSCM',
  branches: [[name: params.commit_sha]],
  userRemoteConfigs: scm.userRemoteConfigs
])
```

(Accessing `scm` fields might require a permission in your setup.)

At this point, the branch specifier can be changed back to `*/master`, so the Jenkinsfile will be checked out from there. Specifying a commit from a branch works now.

![success]({{ site.url }}/assets/images/jenkins-commit/success.png)

## Summary

Like in this case, the root cause is usually the user, but the plugin did a really good job of not helping. This behavior is described as
`If the remote URL does NOT end with /.git, a bare repository is assumed.` in the [documentation](https://www.jenkins.io/doc/pipeline/steps/workflow-scm-step/).
At the end, everything did exactly what it said it would:
- the main SCM configuration checked out the Jenkinsfile only, thus:
![script-from-scm]({{ site.url }}/assets/images/jenkins-commit/script-from-scm.png)
- the Git plugin worked on a local, bare repository as the omitted field doesn't end with `.git` because it's, well, empty


Things like this can be easily overlooked, especially because the code checkout part is usually done once and never touched again.
The basics will always find a way to you, when you least expect them.
