---
title: "Installing pg_repack on AL2023"
description: 'This blog artice describes how to install the pg_repack binary on EC2 using AL2023'
pubDate: 'Jul 05 2025'
image: '/dusty_server.webp'
tags:
  - AWS
  - PostgreSQL
  - Postgres
  - pg_repack
  - install
---

What follows is my journey through the jungle that is postgres bloat removal. If you just want to know how to install
pg_repack on AL2023, skip to [this section](#how-to-install-pg_repack-on-al2023)

## Some Context

I recently found myself in need of deleting a large portion of rows of a database table. In fact, it was millions of
entries that made the table the second largest in the database, with well over 500 GB of storage space consumed.
After the deletion was done, I was surprised that the storage space was still occupied. A little bit of digging
revealed that postgres does in fact NOT delete rows directly. Deleted rows are first marked as "dead" and it requires
a `VACUUM` operation to clean these dead rows up.
To my surprise, however, `VACUUM` does not release storage space to the OS directly. Instead, dead rows are cleaned up
and the freed-up space re-used by new rows in the same table.
To "shrink" the physical table size, a `VACUUM FULL` is required, which in turn requires an `ACCESS EXCLUSIVE LOCK` for the
whole table. A pity, because that would mean that any query on that table would need to wait until that VACUUM FULL
operation is done. That's not an option for production-level workloads.

A little bit of further research brought me to this blog post from AWS which proposes to use the `pg_repack` extension
of postgres as an alternative to `VACUUM FULL`, as it does not need a long-lived `ACCESS EXCLUSIVE LOCK`.
Link: https://aws.amazon.com/de/blogs/database/remove-bloat-from-amazon-aurora-and-rds-for-postgresql-with-pg_repack/

Reading through it, it seemed almost too easy to do. Just a simple `create extension pg_repack;` on the database,
then finding out what version of pg_repack was installed (1.5.0 in my case) and finally I stumbled across this
prerequisite:

    You need to have a client machine, for example an EC2 instance with pg_repack installed#
    in it and connectivity to your RDS or Aurora instance.

I thought to myself: _great, how hard could it be. Simply install a dependency on EC2_...
What came after that was a 6-hour-long journey of figuring out how to install `pgxn`, postgres development libs and
finally `pg_repack` on AL2023.

## Setting up the EC2 instance

This should be as simple as following any tutorial on the web. A word of caution, though. I tried for hours to
use Amazon Linux 2 for this, but that simply does not work. Additionally, Amazon Linux 2 is already deprecated
and not supported anymore, so please do not use it for anything anymore and migrate your workloads.

Here are some tutorials on how to set up the EC2 with AL2023.
* [Getting started with EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
* [AL2023 on EC2](https://docs.aws.amazon.com/linux/al2023/ug/ec2.html#launch-from-ec2-console)
* [Getting started with Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started.html)

## How to install pg_repack on AL2023

Make sure to look up the version of postgres you want to clean up using `pg_repack`. This matters, as you need
different dependencies for each postgres version. I used version `16`, but that can be swapped out easily to any
version above at least `>=10`.

Once the EC2 instance is up and running, and you connected to it via session manager (or any other way),
start a bash via `bash` and elevate your access to `root` via `sudo -s` to execute the following commands:

```shell
python3 -m ensurepip; # python3.9 comes pre-installed on AL2023
pip3 install pgxnclient; # see https://pgxn.github.io/pgxnclient/install.html
dnf install postgresql16-server-devel postgresql16-static lz4-devel readline-devel zlib-devel
pgxn install 'pg_repack=1.5.0' # replace 1.5.0 with whatever version you require
```

That's already it. You should now be able to use `pg_repack --version` to verify the installation worked.
Happy cleaning of databases!


