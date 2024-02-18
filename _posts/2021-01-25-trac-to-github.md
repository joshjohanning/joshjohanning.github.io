---
title: 'So You Want to Migrate Trac Tickets to GitHub Issues'
author: Josh Johanning
date: 2021-01-25 8:30:00 -0600
categories: [GitHub, Migrations]
tags: [GitHub, GitHub Issues, Migrations]
---

## Disclaimer

I should first off state that I wouldn't entirely recommend this, if you are migrating to GitHub for the first time, you should try to start off with a blank slate. Keep the old tickets in Trac or the database around for a period in time in case you need to reference, but don't migrate the entire Trac ticket repository. I could understand wanting to port in active tickets, though, which is a very valid use case for a tool like I am going to demonstrate.

We were working with a client who was migrating off of their old Trac server, and I wanted to document (mostly for myself, but for anyone else who finds this too) how exactly it worked.

## Tools

There are plenty of tools out there on GitHub ([svigerske/trac-to-github](https://github.com/svigerske/trac-to-github), [robertoschwald/migrate-trac-issues-to-github](https://web.archive.org/web/20200912010021/https://github.com/robertoschwald/migrate-trac-issues-to-github), [hershwg/github-migrate-trac-tickets](https://github.com/hershwg/github-migrate-trac-tickets)... some of them require [XML-RPC](https://trac-hacks.org/wiki/XmlRpcPlugin), which I had a heck of a time installing on my Apache Trac webserver, so I wasn't able to test those.

The two I have tested are:

* [mavam/trac-hub (joshjohanning/trac-hub)](https://github.com/mavam/trac-hub)
* [trustmaster/trac2github](https://github.com/trustmaster/trac2github)

Both of these are more focused on Issues, and neither really do attachments (see trac-hub how to section [below](#trac-hub-how-to) for a possible solution though). The [svigerske/trac-to-github](https://github.com/svigerske/trac-to-github) tool mentions that it uploads attachments as gists.

Note: These tested against GitHub's Cloud instance (not server).

## trac-hub

### trac-hub Overview

I originally tested out trac2github and was going to write about that, but I think I might like this [trac-hub](https://github.com/joshjohanning/trac-hub) tool a little better. When comparing the two, trac-hub might be better in that it uses the [Import Issues API](https://gist.github.com/jonmagic/5282384165e0f86ef105). The issues import API never left preview, so be aware it's not officially supported from GitHub. The benefits of using this API are that it can create issues:

* without hitting abuse detection warnings and getting blocked
* without sending email notifications
* without increasing your contribution count to ridiculous heights
* much faster than with the normal issues API
* with correct creation/closed date set
* atomically without users being able to interfere in the creation of a single issue

Basically, using this tool allows the GitHub issue to look like it was created originally when it was created in Trac versus looking like a brand new issue that was created.

[I created a fork of mavam/trac-hub (joshjohanning/trac-hub)](https://github.com/joshjohanning/trac-hub) (that has since been [merged](https://github.com/mavam/trac-hub/pull/33)) that adds the ability to map Trac ticket owners to GitHub Issues assignees. Make sure not to typo the GitHub username as the issue will fail to create if a ticket's owner (assignee) has a mapping in the configuration file. If a mapping doesn't exist, the GitHub Issue will be created with no assignee, as expected.

The caveats of both of the tools is that the "Issue creator" will appear as the one who originally ran the tool - but at least with this tool, we can preserve create and comment dates.

### trac-hub How To

I ran this on a debian 10.7 server, the same server that was running my Trac installation.

Pre-requisites to install:

* `sudo apt-get install bundler`
* `sudo apt-get install libmariadb-dev`
* `sudo apt-get install libsqlite3-dev`

Instructions:

1. Clone the repository - `git clone https://github.com/joshjohanning/trac-hub`
1. Rename the example configuration file - `mv config.yaml.example config.yaml`
1. Edit the configuration file `vim config.yaml` and modify the following sections:
    1. `trac` to provide the sqlite database path or mysql connection (untested)
    1. `github` to provide the target org/repo name and personal access token (make sure to grant the token enough access!)
    1. `users` to provide a list of mapping of Trac users --> GitHub. If a mapping doesn't exist, it won't use the GitHub handle and just refer to the user as the Trac user. In my version of the tool, if a mapping is found but the GitHub handle doesn't exist, it won't migrate tickets --> issues that are owned by/assigned to that non-existant user. Essentially, just make sure that the mapping uses a valid GitHub user or don't include it :)
    1. `labels` to provide a mapping of Trac ticket metadata to [labels in GitHub Issues](https://docs.github.com/en/github/managing-your-work-on-github/managing-labels)
1. Run the migration in a test repo! `sudo bundle exec trac-hub -v -s 1 -F`.  Command line options:
    1. `-v` : verbose/debug logging
    1. `-s 1` : start at the the first ticket in Trac
    1. `-F` : fast import, import without safety-checking the issue number. The way this tool runs is it expects Trac ticket #1 to be created as Issue #1. If you already have issues in the repository, or you want to do testing, add the -F import. For your real migration, you could probably drop this and trac ticket #1 will map to issue #1. Otherwise, with this `-F` argument, the issues will be created even though the ID's won't make 1:1.
    1. No example here, but `-a` for attachment-urls is interesting. The tool doesn't migrate attachments, but if you used one of the `download-trac-attachment-*.sh`{: .filepath} scripts [in the repo](https://github.com/joshjohanning/trac-hub/tree/master/tools), you could host the files somewhere, presumably with the same file names and it will hyperlink the attachments??

### trac-hub Command Line Argument List

```shell
$ bundle exec trac-hub --help
    -c, --config config              set the configuration file
    -s, --start-at ID                start migration from ticket with number <ID>
    -r, --rev-map-file FILE          allows to specify a commit revision mapping FILE
    -a, --attachment-url URL         if attachment files are reachable via a URL we reference this here
    -S, --single-post                Put all issue comments in the first message.
    -F, --fast-import                Import without safety-checking issue numbers.
    -o, --opened-only                Skips the import of closed tickets
    -v, --verbose                    verbose mode
```

### trac-hub Example Migration Screenshots

Migrating 5 tickets from Trac:
![trac-tickets](/assets/screenshots/2021-01-26-trac-to-github/trac-tickets.png){: .shadow }
_Original ticket list in Trac_

How they appear in GitHub:
![trac-hub-example-issues](/assets/screenshots/2021-01-26-trac-to-github/trac-hub-example-issues.png){: .shadow }
_Issues in GitHub migrated using trac-hub_

Trac ticket comments:
![trac-ticket-comments](/assets/screenshots/2021-01-26-trac-to-github/trac-ticket-comments.png){: .shadow }
_Original ticket comments in Trac_

Issue comments:
![trac-hub-example-issue-comments](/assets/screenshots/2021-01-26-trac-to-github/trac-hub-example-issue-comments.png){: .shadow }
_Issue comments in GitHub migrated using trac-hub_

## trac2github

### trac2github Overview

This was the first [tool](https://github.com/trustmaster/trac2github) that I used, and it works! It uses the regular GitHub Issues API, which is subject to rate limits and abuse detection. The team that tried to use this import with me ran into the abuse detection a few times and tweaked lines [484](https://github.com/trustmaster/trac2github/blob/master/trac2github.php#L484) and [485](https://github.com/trustmaster/trac2github/blob/master/trac2github.php#L485) of the php script to play more friendly with GitHub (decreasing `$request_count > 50)` and increasing `sleep(70)`. Note that these are only used if `$ticket_limit` is set in the [configuration file](https://github.com/trustmaster/trac2github/blob/master/trac2github.cfg#L68).

The team did not like this tool because the original create date was not preserved - all of the issues look like they were created at the time the import tool ran.

Otherwise, it does the job at migrating tickets, comments, user mapping with assignee, and label mapping.

### trac2github How To

I ran this on a debian 10.7 server, the same server that was running my Trac installation.

Pre-requisites to install:

* PHP (e.g. on Ubuntu/Debian `sudo apt-get install php`) (I prefer `sudo apt-get install php-fpm` to NOT install a new version of apache over my existing web server)
* Support for the trac database format, e.g. `sudo apt-get install php-mysql`, `sudo apt-get install php-sqlite3`, etc.
* `sudo apt-get install php-curl`

Instructions:

1. Clone the repository - `git clone https://github.com/joshjohanning/trac-hub`
1. Edit the configuration file `vim config.yaml` and modify the following variables:
    1. `$username` : GitHub username
    b. `$password` GitHub personal access token (make sure to grant the token enough access!)
    c. `$project` : The GitHub organization you are migrating to. If migrating issues to your own GitHub repo under your own account, this would be the same as `$username`. Use the Organization name if using GitHub Enterprise
    1. `$repo` : The GitHub repo name
    1. `$users_list` : Provide a mapping of Trac user --> GitHub user
    1. The database driver settings - whether that's `$mysqlhost_*`, `$sqlite_trac_path`, or `$pgsql_` settings.
    1. Explore the [other items in the config file](https://github.com/trustmaster/trac2github/blob/master/trac2github.cfg) to see if they are needed, such as `$ticket_offset` for resuming a migration or `$remap_labels` to modify the label mapping. Note that while you might not think you need to set `$ticket_limit` because you want to migrate the entire Trac ticket database, this setting needs to be set in order to trigger the [aforementioned](#trac2github-overview) rate limiting sleep control. **Therefore, I advise giving `$ticket_limit` an arbitrary value for this purpose.**
1. Run the import: `php trac2github.php`

You'll notice that this import is a lot slower than trac-hub :).

### trac2github Example Migration Screenshots

Migrating 5 tickets from Trac:
![trac-tickets](/assets/screenshots/2021-01-26-trac-to-github/trac-tickets.png){: .shadow }
_Original ticket list in Trac_

How they appear in GitHub:
![trac-hub-example-issues](/assets/screenshots/2021-01-26-trac-to-github/trac2github-example-issues.png){: .shadow }
_Issues in GitHub migrated using trac2github_

Trac ticket comments:
![trac-ticket-comments](/assets/screenshots/2021-01-26-trac-to-github/trac-ticket-comments.png){: .shadow }
_Original ticket comments in Trac_

Issue comments:
![trac-hub-example-issue-comments](/assets/screenshots/2021-01-26-trac-to-github/trac2github-example-issue-comments.png){: .shadow }
_Issue comments in GitHub migrated using trac2github_

## Takeaways

Run a few of these migrations, and run them in repositories that you don't mind deleting afterwards as deleting issues can be...challenging.

Run the migration using someone's PAT that you don't mind being the creator for the Issues (throwaway / service account possibly??).

Make sure you have the users created in GitHub and mapped in the appropriate configuration file ahead of time.

Be patience, have reasonable expectations, and good luck!
