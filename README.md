# Why does SELinux love containers?
I have been dealing with SELinux for years, and have often been asked or told that SELinux is difficult to use, and how come I don't make it easier?  My response is always that SELinux reveals the complexity of the Operating System.  

SELinux is all about controlling communications paths between two processes. If a compromised process can interact with another process, then this is a route for the compromised application to take over the other process or steal its content.

The problem is as Unix and Linux evolved, applications have developed multiple ways to communicate as well as multiple places to be installed. 

For example you might have a Web front end and a database back end on the same system. Say the Database contains Credit Card data and the Web front end prompts for the users Credit Card number.  When the user logs in with the correct password, the Database CC number can be used to complete the 
transaction.  The way this is supposed to work is the username and password
are handed to the database and the credit card is returned. If the Web front end was compromised then the compromised processes would want to get to the database of credit card data.

##### SELinux is a labeling system
##### Every process has a label
##### Every file, directory, file system object gets a label
##### Policy governs interaction between processes labels and file labels
##### The kernel enforces the rules

Lets look at how many different ways these labels can be setup on a modern Linux system.  Here are the potential web front ends as defined in targeted policy.
```
# grep httpd_exec_t /etc/selinux/targeted/contexts/files/file_contexts
/usr/sbin/httpd(\.worker)?	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/apache(2)?	--	system_u:object_r:httpd_exec_t:s0
/usr/lib/apache-ssl/.+	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/apache-ssl(2)?	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/nginx	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/thttpd	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/php-fpm	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/cherokee	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/lighttpd	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/httpd\.event	--	system_u:object_r:httpd_exec_t:s0
/usr/bin/mongrel_rails	--	system_u:object_r:httpd_exec_t:s0
/usr/sbin/htcacheclean	--	system_u:object_r:httpd_exec_t:s0
```
Databases (Just mysql, postgresql, and mongodb for this example)
```
# grep -e mysqld_exec_t -e mongod_exec_t -e postgresql_exec /etc/selinux/targeted/contexts/files/file_contexts
/usr/bin/(se)?postgres	--	system_u:object_r:postgresql_exec_t:s0
/usr/bin/initdb(\.sepgsql)?	--	system_u:object_r:postgresql_exec_t:s0
/usr/sbin/mysqld(-max)?	--	system_u:object_r:mysqld_exec_t:s0
/usr/lib/postgresql/bin/.*	--	system_u:object_r:postgresql_exec_t:s0
/usr/sbin/ndbd	--	system_u:object_r:mysqld_exec_t:s0
/usr/bin/mongod	--	system_u:object_r:mongod_exec_t:s0
/usr/bin/mongos	--	system_u:object_r:mongod_exec_t:s0
/usr/bin/pg_ctl	--	system_u:object_r:postgresql_exec_t:s0
/usr/libexec/mysqld	--	system_u:object_r:mysqld_exec_t:s0
/usr/bin/mysql_upgrade	--	system_u:object_r:mysqld_exec_t:s0
/usr/libexec/postgresql-ctl	--	system_u:object_r:postgresql_exec_t:s0
/usr/libexec/mongodb-scl-helper	--	system_u:object_r:mongod_exec_t:s0
/usr/lib/pgsql/test/regress/pg_regress	--	system_u:object_r:postgresql_exec_t:s0
/usr/share/aeolus-conductor/dbomatic/dbomatic	--	system_u:object_r:mongod_exec_t:s0

```
Now if we just look for how many file context specifications there are for httpd, mongo, mariadb and postgresql.

```
grep -e httpd -e mysql -e mongo -e postgres /etc/selinux/targeted/contexts/files/file_contexts | wc -l
248
```
That means that there are 248 different file paths where content for a database or web front end might be stored.  

### It gets worse.
Administrators can decide they don't want their content to be stored in these locations, so they want to put their database data in /company/db/data.db.  When the admin does this and the database blows up he will need to execute commands like the following
```
semanage fcontext -a -t mysqld_db_t '/company/db(/.*)?'
restorecon -R -v /company
```

Now the web front end may communicate with the databases using any of the following:

* Unix Domain Sockets
* Shared Memory
* Dbus
* Network
* Abstract Sockets
* Others?

If the administrator decides to pick one that the policy writers did not know about or puts the UNIX domain sockets in a directory that we did not know about SELinux will block the communications.


The way I think of SELinux in a modern linux system is trying to keep two amoebas from touching each other.

IMAGE OF amoebas.

## Containers to the rescue.

Containers (micro services) greatly simplify the Operating System from an SELinux point of view. 

In a container world you would run your web front end in one container and your database in a separate container.  Containers define that the only way for two containers to communicate is via the network.

With docker containers we run all of the container processes with a single type, `svirt_lxc_net_t`.  Docker content is all labeled with a single type `svirt_sandbox_file_t`.
Policy rules say that the only type that a `svirt_lxc_net_t` process is allowed to write is `svirt_sandbox_file_t`.  We take advantage of MCS separation to isolate the containers from each other.  Docker and other container technology tools setup the processes to run with the correct label and the content to be labeled correctly.  They also pick out a unique MCS label for each container to make sure they are not allowed to interact except via the network.  We can use the same policy for the web front end and for the database.  In the container world, we can run all confined containers with the same type and label the content the same.

Only issues which come up with docker and SELinux is around volume mounting.  If you want to volume mount a directory from the host into a container, you have to make sure it is labeled correctly, but even that is easy.  Docker supports two SELinux options to the --volume command. 

* Z indicates the volume will be private to the container
* z indicates the volume can be shared amongst other containers.

If the user executes the following
```
docker run -d -v /var/lib/mysql:/var/lib/mysql:Z mariadb
```
Docker will automatically setup the labeling of /var/lib/mysql on the host to be privately used within this container.

Running containers allows us to treat all of the containers as a black box and and able to control just the network connections between the boxes.

## seandroid - SELinux for android phones

Android phones from Google and Samsung and probably several others have been taking advantage of SELinux for several years without people clamoring for `setenforce 0`, because Apps in Android are pretty much the equivalent of container applications.

Image of phone with SELinux/SEandroid on.

## A new policy
We have a started an effort to write a brand new policy for the Atomic platform.
We are thinking of setting up a greatly simplified policy with few types and just isolate processes on the system running as say `systemd_t`.  Then have processes running in side of container running as container_t.

https://wiki.brq.redhat.com/SecurityTechnologies/AdvancedContainersHostingPlatformsIsolation

## SELinux is critical for container separation

Running `setenforce 0` on a container platform should be considered hightly dangerous, and luckily container technology has made SELinux easier to deal with.
