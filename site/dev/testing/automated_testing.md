Skia Automated Testing
======================

Overview
--------

Skia uses [Swarming](https://github.com/luci/luci-py/blob/master/appengine/swarming/doc/Design.md)
to do the heavy lifting for our automated testing. It farms out tasks, which may
consist of compiling code, running tests, or any number of other things, to our
bots, which are virtual or real machines living in our local lab, Chrome Infra's
lab, or in GCE.

The [Skia Task Scheduler](http://go/skia-task-scheduler) determines what tasks
should run on what bots at what time. See the link for a detailed explanation of
how relative task priorities are derived. A *task* corresponds to a single
Swarming task. A *job* is composed of a directed acyclic graph of one or more
*tasks*. The job is complete when all of its component tasks have succeeded
or is considered a failure when any of its component tasks fails. The scheduler
may automatically retry tasks within its set limits. Jobs are not retried.
Multiple jobs may share the same task, for example, tests on two different
Android devices which use the same compiled code.

Each Skia repository has an infra/bots/tasks.json file which defines the jobs
and tasks for the repo. Most jobs will run at every commit, but it is possible
to specify nightly and weekly jobs as well. For convenience, most repos also
have a gen_tasks.go which will generate tasks.json. You will need to
[install Go](https://golang.org/doc/install). From the repository root:

	$ go get -u go.skia.org/infra/...
	$ go run infra/bots/gen_tasks.go

It is necessary to run gen_tasks.go every time it is changed or every time an
[asset](https://skia.googlesource.com/skia/+/master/infra/bots/assets/README.md)
has changed. There is also a test mode which simply verifies that the tasks.json
file is up to date:

	$ go run infra/bots/gen_tasks.go --test



Try Jobs
--------

It is useful to know how your change will perform before it is submitted. After
uploading your CL to [Gerrit](https://skia-review.googlesource.com/), you may
trigger a try job for any job listed in tasks.json:

	$ git cl try -B <bucket name> -b <job name>

The bucket name refers to the [Buildbucket](https://chromium.googlesource.com/infra/infra/+/master/appengine/cr-buildbucket/README.md)
bucket to which the request will be submitted. Most public Skia repos use the
"skia.primary" bucket, and most private Skia repos use the "skia.internal"
bucket.


Status View
------------

The status view shows a table with tasks, grouped by test type and platform,
on the X-axis and commits on the Y-axis.  The cells are colored according to
the status of the task for each commit:

* green: success
* orange: failure
* purple: exception (infrastructure issue)
* black border, no fill: task in progress
* blank: no task has started yet for a given revision

Commits are listed by author, and the branch on which the commit was made is
shown on the very left. A purple result will override an orange result.

For more detail, you can click on an individual cell to get a summary of the
task.  You can also click one of the white bars at the top of each column to see
a summary of recent tasks with the same name.

The status page has several filters which can be used to show only a subset of
task specs:

* Interesting: Task specs which have both successes and failures within the
  visible commit window.
* Failures: Task specs which have failures within the visible commit window.
* Comments: Task specs which have comments.
* Failing w/o comment: task specs which have failures within the visible commit
  window but have no comments.
* All: Display all tasks.
* Search: Enter a search string. Substrings and regular expressions may be
  used, per the Javascript String Match() rules:
  http://www.w3schools.com/jsref/jsref_match.asp

<a name="adding-new-jobs"></a>
Adding new jobs
---------------

If you would like to add jobs to build or test new configurations, please file a
[New Bot Request][new bot request].

If you know that the new jobs will need new hardware or you aren't sure which
existing bots should run the new jobs, assign to jcgregorio. Once the Infra team
has allocated the hardware, we will assign back to you to complete the process.

Generally it's possible to copy an existing job and make changes to accomplish
what you want. You will need to add the new job to
[infra/bots/jobs.json][jobs json]. In some cases, you will need to make changes
to recipes:

* If there are new GN flags or compiler options:
  [infra/bots/recipe_modules/flavor/gn_flavor.py][gn flavor py]
* If there are modifications to dm flags: [infra/bots/recipes/test.py][test py]
* If there are modifications to nanobench flags:
  [infra/bots/recipes/perf.py][perf py]

After modifying any of the above files, run `make train` in the infra/bots
directory to update generated files. Upload the CL, then run `git cl try -B
skia.primary -b <job name>` to run the new job. (After commit, the new job will
appear in the PolyGerrit UI after the next successful run of the
Housekeeper-Nightly-UpdateMetaConfig task.)

If you need to do something more complicated, or if you are not sure how to add
and configure the new jobs, please ask for help from borenet, benjaminwagner, or
mtklein.

[new bot request]:
    https://bugs.chromium.org/p/skia/issues/entry?template=New+Bot+Request
[jobs json]: https://skia.googlesource.com/skia/+/master/infra/bots/jobs.json
[gn flavor py]:
    https://skia.googlesource.com/skia/+/master/infra/bots/recipe_modules/flavor/gn_flavor.py
[test py]:
    https://skia.googlesource.com/skia/+/master/infra/bots/recipes/test.py
[perf py]:
    https://skia.googlesource.com/skia/+/master/infra/bots/recipes/perf.py
