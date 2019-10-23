# Lab 6 demo 1: backup tools

## Intro

There are a lot of ways how to manage SQL backups. This demo will show only a
few of them, not necessarily the best ones -- but should give you some basic
understanding of how backups work.

First, we will set up a primitive backup system using `mysqldump` and `scp`.

Then we will improve it using more advanced tools: `rsync` and `duplicity`.

Finally, we will automate our backups with Crontab.

All commands shown below should be run as user `backup` if not stated otherwise.

This demo is by no means an example of a complete backup system. Its goal is to
illustrate _some_ of the aspects of a backup process.


## Setup

For this demo we will need to set up a few services.

`backup-server` instance with:
 - User `backup` created
 - Directory `/srv/backup/db-server` created and writable for user `backup`

`db-server` instance with:
 - User `backup` created
 - Directory `/var/backup` created and writable for user `backup`
 - Working SSH connection to `backup-server` for user `backup`
 - MySQL server running
 - MySQL database `demo` created
 - MySQL user `backup` created with all privileges in database `demo`

We will a table called `demo` in the database `demo` with the following content:

    +----+------+
    | id | val  |
    +----+------+
    |  1 | 42   |
    |  2 | 1337 |
    +----+------+

These database and table can be created using these SQL commands:

    DROP TABLE IF EXISTS demo;
    CREATE TABLE demo (
        id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
        val VARCHAR(31)
    );
    INSERT INTO demo (val) VALUES (42), (1337);

We will also need to configure passwordless access for user `backup` _for
demonstration purposes_.

**Passwordless access for `backup` user is not a bright idea for
production services!**


## Extract data from the existing database

On a `db-server` we should have MySQL server running and database called `demo`
created with a single table called `demo`. You can check the table contents by
running this command as user `backup`:

    mysql -e 'SELECT * FROM demo.demo'

You should see something like this:

    +----+------+
    | id | val  |
    +----+------+
    |  1 | 42   |
    |  2 | 1337 |
    +----+------+

This is the initial state of out database.

You can extract (called 'dump' in database world) the data from running MySQL
database with this command:

    mysqldump demo

This is the data we will need to backup, and this data should be sufficient to
restore the database.

Save the MySQL dump to a file:

    mysqldump demo > /var/backup/demo.sql
    less /var/backup/demo.sql

That's it. We now have a dump of the latest state of MySQL database `demo`, and
a simple script to create another dump when needed.

We will now need to upload the backup to the backup server.


## SCP

SCP, or [Secure copy protocol](https://en.wikipedia.org/wiki/Secure_copy) is the
simplest way to transfer files between two different machines.

You can upload the MySQL dump to backup server with one simple command:

    scp /var/backup/* backup-server:/srv/backup/db-server/

First argument (`/var/backup/*`) is the list of source files to copy, and the
second one (`backup-server:/srv/backup/db-server/`) is destination directory on
the remote machine.

This will upload the previously created `demo.sql` file to the backup server
over SSH. The dump file copy should now appear on the backup server:

    less /srv/backup/db-server/demo.sql

So the '2-1' part of the backup rule '3-2-1' is implemented: you have two copies
of the data stored locally (on site).

However `scp` might be not the best choice for larger amounts of data.

For the single SQL table with just a few lines it may be enough, but for real
life cases with lots of larger files `scp` has a lot of limitations. For
instance, it will upload the entire file every time. Try running the same
upload command again, multiple times:

    scp /var/backup/* backup-server:/srv/backup/db-server/
    scp /var/backup/* backup-server:/srv/backup/db-server/

Note that the number of bytes uploaded (column 3) is the same every time, and is
equal to the actual size of the file:

    ls -la /var/backup/demo.sql

What if we could just upload the difference between files to save the time and
network bandwidth? This approach is called 'synchronization', or 'sync' for
short, and there is a tool for that.


## Rsync

[Rsync](https://rsync.samba.org/) is a protocol and simple utility for
synchronizing files between different directories on one computer, and also
different computers over the network. We will use it to upload our MySQL dump
file to the backup server as we previously did with `scp`:

    rsync -rv /var/backup/* backup-server:/srv/backup/db-server

Same logic here: first argument is the list of files to copy, and the second one
is the destination directory.

Note that amount of bytes transferred is much less than the file size:

    ...
    sent 106 bytes  received 53 bytes  318.00 bytes/sec
    total size is 1,892  speedup is 11.90

This is happening because `rsync` is not uploading the entire file but only the
differences; in this case the file is the same in source (DB server) and
destination (backup server), so no actual data chunks are sent at all; only a
few bytes (106 sent, 53 received in this example) are spent of figuring out the
difference.

Let's now simulate a live system behavior and add some data to the database:

    mysql -e 'INSERT INTO demo.demo (val) VALUES (9001)'
    mysql -e 'SELECT * FROM demo.demo'

Run your backup script again to extract the latest data:

    mysqldump demo > /var/backup/demo.sql

And run the `rsync` again to sync the dump to backup server:

    rsync -rv /var/backup/* backup-server:/srv/backup/db-server/

This command should print something similar to:

    ...
    sent 1,304 bytes  received 53 bytes  904.67 bytes/sec
    total size is 1,903  speedup is 1.40

The amount of bytes transferred is now bigger, obviously some changes were sent.
But the sent changes together with `rsync` own protocol payload (1304 bytes) are
still smaller than the actual file size.

The win in this example may seem insignificant, but imagine that you have 100 GB
database dumps, and only small part of the actual data is changing between
dumps. You will clearly win a lot of network bandwidth while using `rsync`
instead of `scp`.

So now we have at least two options how to upload the backups to the backup
server. But are these backups usable? Let's try to restore them to find out.

For things to look more real let's actually destroy the table first, and also
delete all local backups to make things more interesting:

    mysql -e 'DROP TABLE demo.demo'
    rm /var/backup/*

Database is gone now, all data is lost:

    mysql -e 'SELECT * FROM demo.demo'
    ERROR 1146 (42S02) at line 1: Table 'demo.demo' doesn't exist

But we have backups! We now need to download the backup to restore the database.
With `rsync` it would just mean syncing the files in the 'other direction', from
backup server to database server:

    rsync -rv backup-server:/srv/backup/db-server/* /var/backup/restore/

Note that the first argument is the source file list _on the remote machine_ in
this case, and the second one is the destination directory on the server that is
being restored.

Also note that restore location is different from the backup location. It is
always a good idea to keep the backup and restore data separately in order not
to destroy the latest backup accidentally. Remember, it is better to have more
copies of the backup than have no copies at all.

Check the downloaded backup:

    less /var/backup/restore/demo.sql

It should be the same SQL file as we previously uploaded to the backup server.
Now let's try to restore the database:

    mysql demo < /var/backup/restore/demo.sql

**Note: this will destroy the current content of `demo` database, if any, and
replace it with the data from the backup!**

Check the content of the `demo` table now; it should be the same as prior to the
last backup:

    mysql -e 'SELECT * FROM demo.demo'

So the backup is usable -- it can be used to restore the service.

There is one significant problem with using `rsync` as a backup tool though. It
cannot store multiple versions of the files being synchronized. To be precise,
`rsync` can either store one ot two versions of the files, but not more.

Let's add another record to our database:

    mysql -e 'INSERT INTO demo.demo (val) VALUES (9002)'
    mysql -e 'SELECT * FROM demo.demo'

... and create another backup:

    rm -rf /var/backup/restore
    mysqldump demo > /var/backup/demo.sql
    rsync -brv /var/backup/* backup-server:/srv/backup/db-server/

Note the `-b` parameter -- this tells `rsync` to create a backup copy of the
file being synced if this file was changed. Check file differences on the backup
server:

    diff -u --color /srv/backup/db-server/*

If you create another version of the backup, the oldest version will be deleted,
and only the last two versions will be preserved:

    mysql -e 'INSERT INTO demo.demo (val) VALUES (9003)'
    mysql -e 'SELECT * FROM demo.demo'
    mysqldump demo > /var/backup/demo.sql
    rsync -brv /var/backup/* backup-server:/srv/backup/db-server

Check the backup directory content on the backup server:

    ls -la /srv/backup/db-server
    diff -u --color /srv/backup/db-server/*

So `rsync` alone does not quite solve the problem if you need more than two
versions of the backup to be stored :(

There are few other things that are essential for backups but cannot be handled
with `rsync` alone:
 - You need data compression to optimize both the storage and bandwidth? Write
   another script.
 - You need to encrypt your backups? Write another script.
 - You need some mechanism to rotate backup versions and delete the old ones?
   Write another script.
 - And so on.

One important thing to keep in mind about `rsync` that it is not a backup tool
-- it is _data synchronization_ tool. This is a good choice to implement the '2'
part of the '3-2-1' backup rule: once you have a repository of usable backups on
your backup server you can use `rsync` to synchronize this repository to some
remote (off-site) location.

But how to build this backup repository in the right way, with multiple version
of the backup stored? Surely there is a tool for that, too.


## Duplicity

[Duplicity](http://duplicity.nongnu.org/) is a specialized backup tool that
operates over Rsync protocol to transfer the data and adds some backup specific
features to that. It is not installed on Ubuntu servers by default, you may need
to install it first:

    sudo apt install duplicity

Duplicity supports incremental backups, backup versioning, rotation and
encryption out of the box. We will skip the encryption part for this demo, but
again, this is only done _for demonstration purposes_.

**Always encrypt your production backups, and make sure to backup the encryption
keys separately!**

Let's create another backup with Duplicity. We have a recent MySQL dump already,
we can reuse it:

    duplicity --no-encryption full /var/backup rsync://backup-server//srv/backup/db-server

Note that we're creating a _full_ backup here. First backup should always be a
full one. Second, third and later could be incremental, but the full backup
should be created again from time to time.

Check the backup directory content on the backup server:

    ls -la /srv/backup/db-server/

Note that three additional files were created:
 - `duplicity-full.20191022T171912Z.manifest`
 - `duplicity-full.20191022T171912Z.vol1.difftar.gz`
 - `duplicity-full-signatures.20191022T171912Z.sigtar.gz`

This solution seems more complicated than a regular `rsync`. So what's the win?
Let's add some more data to our database and create another backup to find out:

    mysql -e 'INSERT INTO demo.demo (val) VALUES (9004)'
    mysql -e 'SELECT * FROM demo.demo'
    mysqldump demo > /var/backup/demo.sql
    duplicity --no-encryption incremental /var/backup rsync://backup-server//srv/backup/db-server

On backup server three more files are created:
 - `duplicity-inc.20191022T171912Z.to.20191022T172048Z.manifest`
 - `duplicity-inc.20191022T171912Z.to.20191022T172048Z.vol1.difftar.gz`
 - `duplicity-new-signatures.20191022T171912Z.to.20191022T172048Z.sigtar.gz`

Let's now inspect what Duplicity is writing to these files:

    zcat duplicity-full.20191022T171912Z.vol1.difftar.gz
    zcat duplicity-inc.20191022T171912Z.to.20191022T172048Z.vol1.difftar.gz

Note that incremental backup contains only the part of SQL dump. Is it broken?
Surely not. Duplicity can use this increment, all previous increments and the
full backup to restore your original SQL dump file.

Also note that incremental file contains much more info than just a diff. This
is happening because Duplicity uses other mechanism to compute the file
differences: `diff` operates on text files while Duplicity computes binary data
blocks of certain length.

So, main question -- is this backup usable? Restoring it is the way to find out!
Destroy the database first:

    mysql -e 'DROP TABLE demo.demo'
    rm -r /var/backup/*
    mysql -e 'SELECT * FROM demo.demo' # this should fail as the table is deleted

... and restore the backup:

    duplicity --no-encryption restore rsync://backup-server//srv/backup/db-server /var/backup/restore
    mysql demo < /var/backup/restore/demo.sql
    mysql -e 'SELECT * FROM demo.demo'

Ta-daa!

Duplicity downloaded needed full backup and increments from the backup server
and assembled the original SQL dump file from these. Note that the file itself
is the same as we backed up:

    less /var/backup/restore/demo.sql

And there are other Duplicity perks as well:
 - Backup encryption using GnuPG is enabled by default; we were disabling it in
   this demo with `--no-encryption` parameter, but in real life you shouldn't;
 - Backup data is compressed to optimize both the storage and network bandwidth;
   `rsync` tool would only care about the latter;
 - There are options to delete old backups -- these can be used in backup
   scripts to keep the backup repository fit;
 - Cloud storage support: you can upload backups directly to AWS S3 and others
   -- you may not even need a backup server, but only as long as you still keep
   following the '3-2-1' rule!

So seems that we have a good enough solution to create the backups manually. But
good backup is run automatically by established schedule. So how to automate it?

Of course, there is a tool for that :)


## Cron

Cron is a time based job scheduler for UNIX systems. It is very widely used, and
your managed system will have it installed by default in most cases.

Cron schedules (called 'crontabs') are stored in /etc/cron.d:

    ls -la /etc/cron.d/*

...and have the following structure:

    <schedule> <user> <shell-command>

`<user>` part is a later addition to some modern Cron implementations;
originally Cron jobs could only be run as `root`.

Cron schedule consists of five fields:

    <minute> <hour> <day-of-month> <month> <day-of-week>

whereas '*' means 'every'. For example, every-minute schedule definition is:

    * * * * *

Daily schedule (01:42 AM) would be defined as

    42 1 * * *

Weekly schedule (02:37 AM every Sunday) would be defined as

    37 2 * * 0

and so on.

You can also define more complex schedules with special characters;
[Wikipedia article on Cron](https://en.wikipedia.org/wiki/Cron#CRON_expression)
is a good starting point.

**Question:** For these Cron schedules, when would the job be run?

    1 2 3 * *

    1 2 3 4 5

    * * 3 4 *

There is a nice online tool to find out: https://crontab.guru.

Crontab is the simplest way to automate you backups: you will just need to write
a script to create a backup, and define a schedule when to run it.

Based on on current setup one possible backup crontab (stored in
`/etc/cron.d/backup` for example) could look like this:

    11 0 * * *    backup  mysqldump demo > /var/backup/mysql.demo.sql
    22 0 * * 0    backup  duplicity full /var/backup rsync://backup-server//srv/backup/db-server
    22 0 * * 1-6  backup  duplicity incremental /var/backup rsync://backup-server//srv/backup/db-server

This would
 - create a fresh MySQL dump nightly at 00:11 AM,
 - create a full backup every Sunday at 00:22 AM, and
 - create an incremental backup every other day (Monday to Saturday) at 00:22 AM

How to verify if your crontab is correct? Wait for 1..2 minutes and check the
system logs:

    tail /var/log/syslog

-- if you made a typo in the crontab you will find something similar there:

    Oct 22 17:39:01 red cron[705]: (*system*backup) RELOAD (/etc/cron.d/backup)
    Oct 22 17:39:01 red cron[705]: Error: bad minute; while reading /etc/cron.d/backup
    Oct 22 17:39:01 red cron[705]: (*system*backup) ERROR (Syntax error, this crontab file will be ignored)

We can add another rule to clean up the backup repository and discard old
backups:

    11 0 * * 0    backup  duplicity delete-older-than 30d rsync://backup-server//srv/backup/db-server

This would delete all backups older than 30 days.

Finally, we can add another rule to the _backup server_ crontab to sync the
backup repository to offsite location periodically:

    11 1 * * *  backup  rsync -v /srv/backup/* some-offiste-backup-server:/srv/backup

**Question:** What is RPO (recovery point objective) of this backup system?
For simplicity let's agree that
 - every data transfer session between servers takes 1 minute, i. e. backup is
   successfully written to remote server exactly 1 minute after the command was
   started on local server, and
 - server clocks are in perfect sync


## Summary

This demo illustrates the usage of a few basic tools (`mysqldump`, `scp`,
`rsync`, `duplicity` and `cron`) in the context of a backup system.

The result is rather a proof of concept for simple services with small amount of
data, but it is certainly not a complete backup system. We are still missing:
 - monitoring -- how can we make sure that backup was created in time and
   successfully?
 - verification -- we know how to verify the backup manually, but ideally it
   should be automated as well
 - proper security measures
 - documentation and so on

Once your system grows out of home-made crontabs and Duplicity scripts there are
lots of tools on the market that you could consider:
 - [BorgBackup](https://www.borgbackup.org/) -- simpler one,
 - [Bacula](https://www.bacula.org/) and [Bareos](https://www.bareos.org/en/) --
   more sophisticated ones,
 - a few proprietary solutions

But none of them will probably work for you as is, out of the box. You will
still need to configure them to do exactly what you need, and for that you
will need a practical knowledge how backup process works internally.

In simpler words, don't even attempt on complex systems until you know how to
set up a backup system with Cron and Duplicity :)
