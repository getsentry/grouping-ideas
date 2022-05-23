# Platform Bundles

Part of Workflow 2.0.

## Summary

The goal of platform bundles is to create a contributable space where language,
platform and ecosystem improvements to Sentry can be contributed.  The goal is
to lower the friction to contribution of tweaks so we can both internally and
externally make quicker adjustments to Sentry.

## Motivation

Today grouping enhancements can be made specific to platforms, but it's rarely
done.  This is because while we now have the capability to make gradual improvements,
the process on the development side is quite cumbersome.  For an individual grouping
improvement one has to a) contribute the modification b) make a judgement call if
this modification requires a new grouping revision c) create a new grouping revision
d) bump the project epoch and e) cut the default over.  As a result we have rarely
introduced a new grouping revision.  More importantly we have not yet used the gradual
upgrade feature except for the switch to the hierarchical grouping EA.

## Detailed Design

This proposal proposes three parts: a change to the workflow through a new repository,
an automation to cut new grouping revisions and additional functionality in the core
grouping system to support this.

### Workflow Changes

We propose a new repository (`getsentry/platform-tweaks`) which contains tweaks to
abstract platforms.  Note that these platforms do not necessarily have to correspond
to platforms in the core system, but are more fine grained.  For instance Vue can have
tweaks independent of JavaScript as a whole.  For now the tweaks are limited to grouping
but longer term it would be preferrable to also add UI specific tweaks for platforms
(such as info links to errors, changing prominence of UI elements etc.).

The layout of the repository is proposed as such:

```
/README.md
/tweaks
  /react                # the name of the tweak is arbitrary
    /detecter.py        # an optional detecter
    /enhancements.txt   # the grouping enhancements
```

Once a configured interval an automated process takes all the latest tweaks from this
repository can compiles a new grouping version from it and commits it / creates a pull
request to the main `getsentry/sentry` repository.

**Detecter File**

The detecter is an optional file which contains a detecter function that given an event
tries to detect what's in it.  It gets an abstact event info struct which is an abstraction
over the real event with the common information pulled out and returns a tuple of tweak-name
and score.

```python
def detect_tweak(event_info):
    if event_info.sdk_name == "sentry.javascript.react":
        return ("react", 1000)
```

**Grouping Enhancement File**

The format of the grouping file looks roughly like this:

```yaml
# ---
# SCORE=1000               # sets the ordering compared to other enhancement files
# ---

# Enhancer rules to apply.  Note that these are *always* loaded so to
# restrict them to only the tweak it needs to be matched explicitly
tweak:react module:**/node_modules/** -app

# example not tweak local
platform:javascript module:**/node_modules/** -app
```

It's likely that the enhancer system might have to gain additional functionality not yet
present.  For instance it likely needs a way to completely remove stack traces, parts of
frames and more.  Potentially the category system can be used for this.

### Automation

The automation takes all the tweaks and cuts a massive enhancements.txt file alongside
a new grouping configuration into a new file and then opens a pull request against
Sentry.  It roughly does these changes:

* commit a new calver versioned enhancements file
* commit a new calver versioned grouping configuration (TBD: changelog notes)
* bump the project config epoch
* set latest epoch as based for the new config
* TBD: schedule the new version as upgrade path for existing projects
* Run all snapshots and accept them (TBD: do we still want the snapshots there?)

### Additional Functionality

With this type of system we likely are going to run into limitations of the current
grouping system that we addressed with individual Python code.