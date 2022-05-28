# Percona XtraDB Cluster 8

An Ansible role that installs, configures and manages Percona XtraDB cluster 8 for EL 8.
This is for either for a single node, or when using 2 nodes or more. The functionality to add an arbiter is included as well.

**Please read this file carefully before deploying this Ansible role**

## Main functionalities

This role is tested for Ansible v2.9 and higher.

 * Percona XtraDB Cluster 8
 * Secured connection by encrypting mysql traffic
 * Bootstrapping a cluster, includes end2end tests
 * Scaling: add or remove hosts from the cluster **with ease**!
 * Add an arbitrator with ease
 * Adds a beautiful backup script
 * Automatically calculates the recommended mysql configuration settings, based on resources
 * Logrotation configuration
 * Adding user defined users
 * Adding user defined databases
 * Idempotent and desired state

## Requirements

 * Brain
 * At least 2G of RAM is recommended, 512MB for the arbiter is sufficient
 * When clustering, the nodes are able to connect to each other via ports 4444, 4567, 4568, see [info](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/faq.html#what-tcp-ports-are-used-by-percona-xtradb-cluster)
 * Plenty of disk space, keep the backup in mind as well. The use of an SSD is preferred for performance
 * Internet connection to download installation files

Copy the variables from `default/main.yml` and adjust them for each node in it's variable folder.

The arbiter requires less configuration, see below.

## Installation

Please first check `defaults/main.yml` for all the variables and to get a good overview.

First, the mysql user and group are set on the system. Then, the dependencies are installed.
Since Percona is installed via the official Percona XtraDB repository, the GPG Keys are added.

The python package [pymysql](https://pypi.org/project/PyMySQL) is required for certain Ansible mysql modules, so this is installed by default.

An override file for the mysql service is placed to set the maximum open files, which is by default 65535. The daemons are reloaded after the change.

Mysql binds by default to `0.0.0.0`.

The variables for passwords can be set in the `group_vars/all.yml`. Ensure to change all variables when running the role.

```yaml
percona_root_password: 'change_me'
percona_system_password: 'change_me'
percona_sst_password: 'change_me'
```

Before starting/restart the mysql service, a config test is executed. If the config check fails, then the role will also fail. This is done in order to maintain a working mysql service. So please test properly on lower environments when adding/removing configurations to mysql.

## Clustering

When there are at least 2 nodes in the play, a multi-master cluster is possible.
A 2 node galera cluster is a fundamentally broken design, as it cannot maintain uptime without a quorum and the ability of a node to go down to aid recovery. Take this into account when designing the cluster.

### Secured connection

By default, the [communication is encrypted](https://www.percona.com/doc/percona-xtradb-cluster/8.0/security/encrypt-traffic.html). As a best practice, the certificates are placed in `/etc/mysql/certs` with proper permissions. The certs in `/var/lib/mysql` are symlinked to `/etc/mysql/certs`.

You should make a choice whether to use encrypted traffic or not before deploying this role. This role is not capable of switching from a 'disabled' certificate state to an enabled state.

When enabled, SSL certificates are used for a secured connection between the hosts. The certificates and keys are fetched from a single host to the controller node and then copied to all the other hosts. After placing, the files are removed from the Ansible controller. As you can imagine, the user executing ansible must have write permissions to the directory where the certs are temporarily stored.

```yaml
percona_ssl: true
# directory to place all keys/certs temporary on the controller node
percona_certs_tmp_dir: /tmp
```

One can disable no_log for debugging purposes with:

```yaml
# set to false to disable no_log
percona_no_log: false
```

### Cluster variables
There must only be one bootstrapper, as this is important, I've created assertions for this matter. An arbiter can also be added, see below.

Ensure the following variables are set for the nodes:

```yaml
# vars for node1, the bootstrapper
percona_cluster:
  name: 'test-cluster'
  enabled: true
  role: master
  bootstrapper: true
  server_id: '1'
  ip_address: 10.0.0.111

# vars for node2
percona_cluster:
  name: 'test-cluster'
  enabled: true
  role: master
  bootstrapper: false
  server_id: '2'
  ip_address: 10.0.0.112
```

### Arbitrator

An arbiter can be included in the cluster, these are the required values for the arbiter, change accordingly:

```yaml
# all vars for node3, the arbiter
percona_cluster:
  name: 'test-cluster'
  enabled: true
  role: arbiter
  bootstrapper: false
  server_id: '3'
  ip_address: 10.0.0.113
```

 * The config file is placed at `/etc/sysconfig/garb`
 * The log file is placed at `/var/log/garbd.log`
 * The process runs as user `root`, an override file is placed for the service

The arbiter also has encrypted traffic, since it makes use of the same certs.

## Scaling

When a host is added to the play, and the vars are added to the new host, the cluster auto scales automagicly! This can be done from a single host to a cluster.

Or from a cluster, to a cluster + 1. ["Do not join several nodes at the same time to avoid overhead due to large amounts of traffic when a new node joins."](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/add-node.html)

The role will, by default, also restart all mysql instances, with a default of a 10s delay between the restarts of each service on a system. See section `systemctl restart mysql` below.

If no restart is desired when scaling, set this variable in `group_vars/all.yml`
Please leave this to `true`, in order to maintain the cluster.

```yaml
percona_cluster_scale_restart_all: false
```

## Login

By default, the user debian-sys-maint is created, with root privileges.
The credentials are set in `/etc/mysql/root.cnf` and is symlinked to `/root/.my.cnf`. One with sudo access can therefore simply execute `mysql` and execute mysql commands.

```bash
[vagrant@test-multi-02 ~]$ sudo mysql
...
mysql>
```

## Backup

The backup script is by default placed in `/etc/mysql/backup.sh`. It uses cron to perform a backup, so crontabs is installed, which is most likely installed by default.

When enabling the backup, ensure there is enough disk space.

To enable the backup functionality, set the required variables:
```yaml
percona_backup:
  enabled: true
  dir: /var/mysql_dump/
  keepdays: '1'
  hour: '05'
  minute: '15'
  minfree: 5 # minimum free disk space in GB to create a backup
  skip:
    databases:
      - skip_this_database_to_backup
    tables:
      - skip_this_table_for_backup
```

The defaults for the mysqldump are:

```bash
[mysqldump]
quick
max_allowed_packet	= 64M
```

These databases are skipped by default when creating a backup:
  * information_schema
  * performance_schema

The backups are stored with a timestamp of that day. A symlink is created to `latest`. The older backup(s) are removed, set this with:

```yaml
percona_backup:
  keepdays: '1'
  ...
```

## Restarting mysql

The role is setup in an idempotent fashion.

Besides the first run, when the role is executed for another run, and a config file has changed, the role will by default restart mysql.

In this case, the mysql service is restarted per host, with a 10s pause in between. This is to prevent all the nodes shutting down at the same time, and losing the cluster.

If mysql fails to start on one node, the role aborts, to maintain the cluster!

If restarting mysql after a config change is undesired, set:

```yaml
percona_ansible_managed: false
```

## Logrotation

There are 2 mysql logs files, placed in /var/log/mysql.

 * Error log, named error.log
 * Slow log, named mysql-slow.log, by default logs queries longer than 3s

There is a logratation configuration placed in `/etc/logrotate.d/`, which is run once a day.
See `templates/logrotate.j2` for the config file.

```yaml
percona_logrotate:
  configure: true
  rotate: 3
```

## Databases and users

Append your own databases and users, set in `group_vars/all.yml`:

```yaml
percona_databases:
  - name: my_first_db
  - name: another_db

percona_users:
  - name: erik
    password: letmein
    privs: '*.*:ALL,GRANT'
    state: present
    encrypted: no
    host: localhost
```

## Mysql values

As stated before, copy all values to a host_vars file, and edit values where applicable.

To include custom keys/values in `mysqld.cnf`:

```yaml
percona_custom_cnf:
  mysqld:
    general-log-file: queries.log
    general-log:
    log-output: file
```

This results in:

```bash
[mysqld]
general-log-file = queries.log
general-log
log-output = file
```

## Example playbook

Ensure to set the required variables in the host_vars of each host.

Single host:

```yaml
---
- hosts:
    - host1
  become: True
  roles:
    - percona_xtradb_cluster
```

A cluster, or scaling from a single node to a cluster:

```yaml
---
- hosts:
    - host1 # master and bootstrapper
    - host2 # master, not bootstrapper
    - host3 # arbiter
  any_errors_fatal: true
  become: True
  roles:
    - percona_xtradb_cluster
```

A cluster, scaling to more hosts:

```yaml
---
- hosts:
    - host1 # master and bootstrapper
    - host2 # master, not bootstrapper
    - host3 # arbiter
    - host4 # master, not bootstrapper
  any_errors_fatal: true
  become: True
  roles:
    - percona_xtradb_cluster
```

## PMM client

It's possible to install the PMM2 client. It will monitor the slow-query, as [this is recommended](https://www.percona.com/doc/percona-monitoring-and-management/conf-mysql.html#id1).

Ensure the pmm server is up and running, and accepting connections.

```yaml
percona_pmm:
  enabled: false
  pkg: pmm2-client
  local_ip: 10.0.0.1
  server_ip: 10.0.0.110
  server_pass: admin
  db_user: pmm
  db_pass: pmm
```

The `(db_pass|db_user)` are created on the mysql database.

## Molecule

There are 2 tests configured with Molecule:

 * Test 1.0 test a single node

 * Test 2.1 setup a multi-master environment, with an arbiter

 * Test 2.2 expand the cluster with another node

Test with:

```bash
molecule test -s default
molecule test -s cluster
```
