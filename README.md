# Bundler cache failure reproduction steps

Claim: `bundle cache` on an UN-installed bundle with
an excluded group can break in a way that UN-breaks when
you run it again.

## Theory of behavior:

The rubygems remote index is constructed and cached in memory (lib/bundler/source/rubygems.rb).

If we do an "install" or similar, the index gets built without caring about "excluded" groups.
If we’re doing “bundle cache” before we’ve done an installation, then we construct a (cached!) rubygems index inside bundler (lib/bundler/source/rubygems.rb) that cares ONLY about the gems we’re installing.   When the caching operation runs to move gems into vendor/cache it does indeed use the same rubygems index (I can tell because my debug monkeys don’t re-fire after it switches from installation to “caching”, so it already has a list of “remote_specs” in it’s instance variable).

Because the excluded dependency wasn’t present during the “install” step when the remote_spec object was constructed, it’s not there to consult at caching time.

However, when you cache AFTER an install has been run (which puts a bunch of stuff on disk) it checks the dependencies it’s going to care about from the bundle definition first, and in “cache” mode that’s all of them (including ones that have been excluded), so at the time the rubygems index gets built (AFTER consulting the spec_set dependency list), there’s a wider set of dependencies to load
which includes the "excluded" gems.  Those gems get loaded into the rubygems index call (and since the index doesn’t exist already, it creates it at the time).

This means that running an install step BEFORE caching solves the problem.

Maybe a `cache` process that has to install should force-dump the rubygems
index from memory before it starts the operation to move gems? Or something?

WORKAROUND: Always `install` before you `cache`.

## Reproduction Steps
0) make sure you're using the config file in
  here, bundle config should look like:

```bash
evizitei@c02zvbkvmd6t repro % bundle config
Settings are listed in order of priority. The top value will be used.
path
Set for your local app (/Users/evizitei/repro/.bundle/config): "bundle"

cache_all
Set for your local app (/Users/evizitei/repro/.bundle/config): true

cache_all_platforms
Set for your local app (/Users/evizitei/repro/.bundle/config): true

no_prune
Set for your local app (/Users/evizitei/repro/.bundle/config): true

build.nokogiri
Set for your local app (/Users/evizitei/repro/.bundle/config): "--use-system-libraries"

without
Set for your local app (/Users/evizitei/repro/.bundle/config): [:pulsar]

evizitei@c02zvbkvmd6t repro %
```

And that you have the latest bundler:

```bash
evizitei@c02zvbkvmd6t repro % bundle --version
Bundler version 2.2.16
evizitei@c02zvbkvmd6t repro %
```

1) wipe out both bundle and vendor:

```bash
rm -rf bundle
rm -rf vendor
```

2) run bundle cache, it installs successfully but
the caching step fails:

```bash
evizitei@c02zvbkvmd6t repro % bundle cache
Fetching gem metadata from https://rubygems.org/.
Fetching rake 13.0.3
Installing rake 13.0.3
Using bundler 2.2.16
Bundle complete! 2 Gemfile dependencies, 2 gems now installed.
Gems in the group 'pulsar' were not installed.
Bundled gems are installed into `./bundle`
Updating files in vendor/cache
Could not find pulsar-client-2.6.1.pre.beta.2 in any of the sources
evizitei@c02zvbkvmd6t repro %
```

3) run bundle cache again, everything works fine

```bash
evizitei@c02zvbkvmd6t repro % bundle cache
Using rake 13.0.3
Using bundler 2.2.16
Bundle complete! 2 Gemfile dependencies, 2 gems now installed.
Gems in the group 'pulsar' were not installed.
Bundled gems are installed into `./bundle`
Updating files in vendor/cache
Fetching gem metadata from https://rubygems.org/.
  * rake-13.0.3.gem
Fetching pulsar-client 2.6.1.pre.beta.2
  * pulsar-client-2.6.1.pre.beta.2.gem
Fetching pulsar-client 2.6.1.pre.beta.2
Fetching pulsar-client 2.6.1.pre.beta.2
Fetching rake-compiler 1.1.1
  * rake-compiler-1.1.1.gem
Fetching rice 2.2.0
  * rice-2.2.0.gem
Fetching rake-compiler 1.1.1
Fetching rice 2.2.0
Fetching rake-compiler 1.1.1
Fetching rice 2.2.0
```