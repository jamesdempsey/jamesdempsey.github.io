---
layout: post
title: "Restoring a single PostgreSQL table from Heroku PGBackups"
date: 2013-07-28 22:16
comments: true
categories: 
---
The following code uses the default Heroku app in the given folder. If there
are multiple apps in the folder, specify the app with `-a APP, --app APP`
where necessary.

Before beginning, it's a good idea to capture a current backup of the
database by running the following command, in case anything goes wrong. The
`--expire` switch deletes the oldest backup.
```sh
$ heroku pgbackups:capture --expire
```

### 1. Get a list of database backups
```sh
$ heroku pgbackups
```

### 2. Download a database dump from a backup
Here, `a405` is the backup name, obtained above.
```sh
$ curl -o mydb.dump `heroku pgbackups:url a405`
```

### 3. Create a local database
You'll need to set up a local database to import the data and then extract the
table.  You can set one up as follows.  The local database name will be used
in the following example as well, if you're using a pre-existing local database.
```sh
$ createdb mydb
```

### 4. Restore the local database from the dump file
Replace the user name and change any other switches as necessary.
```sh
$ pg_restore --verbose --clean --no-acl --no-owner -h localhost -U myuser -d mydb mydb.dump
```

### 5. Dump the table
Selectively dump the table by running the following command specifying the table
name in the `-t mytable` switch.
```sh
$ pg_dump -Fc --no-acl --no-owner -h localhost -U myuser -t mytable mydb > mytable.dump
```
You can find the table name by opening the database in the PostgreSQL terminal
```sh
$ psql mydb
```
```sql
mydb=# \dt
```
where `\dt` outputs a list of all database table names (relations).

### 6. Upload the table dump file
In order to import the dump file into the database on Heroku,
[Heroku requires that the dump file be uploaded somewhere with an HTTP-accessible URL](https://devcenter.heroku.com/articles/heroku-postgres-import-export#import-to-heroku-postgres).
They recommend Amazon S3, so upload the the `mytable.dump` dump file of the
database table to an S3 bucket using a tool such as
[Cyberduck](http://cyberduck.ch/).

### 7. Restore the table dump into the database
Heroku will ask you to confirm by entering the app name.  The default `DATABASE`
should work.  Tables not included in the backup should not be affected.
```sh
$ heroku pgbackups:restore DATABASE 'http://app.s3.amazonaws.com/app/path/to/mytable.dump'
```

As noted at the beginning, if anything goes wrong and you need to restore from
the most recent backup, that can be accomplished with the following command,
where `a405` is the name of the backup obtained with the command in step 1:
```sh
$ heroku pgbackups:restore DATABASE a405
```

You probably don't want that data lying around locally, especially if it's
production data, so drop the local database and delete the dump files locally
and from the S3 bucket.
```sh
$ dropdb mydb
$ rm -P mydb.dump
$ rm -P mytable.dump
```
***
### Sources:
* [Heroku PG Backups](https://devcenter.heroku.com/articles/pgbackups)
* [Importing and Exporting Heroku Postgres Databases with PG Backups](https://devcenter.heroku.com/articles/heroku-postgres-import-export)
* [Cyberduck](http://cyberduck.ch/)
* [Restoring a single table from a Postgres database or backup](http://feeding.cloud.geek.nz/posts/restoring-single-table-from-postgres/)
* [Upload only one table using heroku pg:transfer instead of whole database](http://stackoverflow.com/questions/16908165/upload-only-one-table-using-heroku-pgtransfer-instead-of-whole-database)
