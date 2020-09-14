![https://www.atlassian.com/dam/jcr:a22c9f02-b225-4e34-9f1d-e5ac0265e543/Confluence@2x-blue.png](https://www.atlassian.com/dam/jcr:a22c9f02-b225-4e34-9f1d-e5ac0265e543/Confluence@2x-blue.png)

# Migrate Confluence 6.4 standalone to 7.3 data-center with confidence

So, you are using Atlassian's Confluence in your organization, and it has been crucial in documenting and maintaining all the IP (intellectual property). The business needs prevent you from taking it down for upgrades. Considering it's critical role, the impact of it's downtime, and the perceived complexity during the possible upgrade prevents anyone from touching it. Now you fear that it may eventually become too old to upgrade. 

Well, we are here to help you with that!

Recently, I undertook a similar project for one of our clients, and I documented all the steps along the way. The document eventually became - what we call - an engineering blog post, and we thought  we can share it with the world, so everyone can benefit from it. This document should make sense of all the steps you are required to perform to upgrade your instance of Confluence.

While working on this project, I noticed that Atlassian's official documentation lacks information about various important tasks, which are critical. For example, Atlassian does not talk about the configuration of the PostgreSQL database server, or the NFS share, or synchronization of time across entire infrastructure. I consider these important to be mentioned in any such guide. A lot of deployments are VM based, and the choice of Virtualization technology, it's overhead, and the configuration of the Virtualization host is also something to be kept under consideration. Encryption-at-rest, i.e. disk encryption is also a factor in performance, as it slows down I/O operations. The network firewalls, antivirus, and other security measures also play their role in performance. In short, many aspects around OS, Network and Security, must be considered, evaluated and configured properly before expecting performance from any of your service, whether it is Atlassian product or not.

**Note:** This guide talks about Confluence, but the general principles apply to Jira as well.

## So, what is the task at hand?

In this guide, I will migrate, upgrade, and convert a Confluence standalone instance running on a single VM to data-center mode running on three nodes/VMs. The database service will be moved to a separate standalone server/VM.

The general straight-forward approach will be:
* Install a fresh instance on Confluence on a new VM - as standalone
* Migrate the old data (DB + attachments) to the new DB server and the new Confluence server respectively
* Upgrade Confluence 6.4.0 (standalone) to Confluence 7.4.3 (standalone)
* Convert Confluence 7.4.3 standalone version to Confluence 7.4.3 data-center version by applying the **data-center license**.

However, sometimes, the client may have a expired - or soon-to-expire - Confluence (standalone) license, and they may not want to renew it; and may want to upgrade/convert to data-center instead, while upgrading the version of Confluence along the way. This is very much understandable, and is do-able. In this case the approach will be:

* Install a fresh instance on Confluence on a new VM - as standalone
* Migrate the old data (DB + attachments) to the new DB server and the new Confluence server respectively
* Convert Confluence 6.4.0 standalone to Confluence 6.4.0 data-center by applying the **data-center license**.
* Upgrade Confluence 6.4.0 (data-center) to Confluence 7.4.3 (data-center)

In fact, **this is the scenario covered in this guide.**

## The infrastructure:
All VMs in this setup run on top of VMware, and run Linux OS. It is OK if you are using a different virtualization technology, or just using bare-metal servers.

### Existing / old setup:
* 1 x VM running Confluence, Jira and PostgreSQL database. (`old-confluence.example.com`)

### New setup (Test & Production):
* 1 x VM for PostgreSQL database (`db.example.com`)
* 3 x VMs for Confluence (`confluence1.example.com`,`confluence2.example.com`,`confluence3.example.com` )
* 1 x NFS share served from a separate/dedicated NFS server (`nfs.example.com`)


# The flight plan:
It is expected that you test the complete plan on some test infrastructure. When successful, **only then** perform the production migration.

## Pre-flight checks/preparation:
* Make sure that you/client/business are aware of the add-ons/plugins which will **not** be upgrade-able, and if the client/business is willing to sacrifice the functionality previously provided by those add-ons/plugins.
* In case your Confluence instance looks up user information from a Jira server (Crowd), then make sure that the new Confluence (test and production) servers' IP addresses are added to the "Jira User Server" on production Jira instance. `Jira -> Settings -> User management -> Jira user server`
* In case your Confluence instance looks up user information from a LDAP/AD server, then make sure that the new Confluence (test and production) servers' IP addresses are allowed to access the LDAP/AD.
* Have a SSL reverse-proxy/load-balancer ready with session affinity and WebSockets support. This will be configured later to point to your new Confluence (test or production) setup.
* Verify sudo/root access on related infrastructure.
* Update OS on all servers
* Preferably disable local/host firewall on any of the servers; or, open necessary ports on host firewalls. 
* Preferably disable SELinux / AppArmor; or, set the right security context for the directories where the Confluence process will read and write files. Remember, SELinux requires a filesystem that supports "security labels", and thus cannot provide access control for files mounted via NFS.
* Since this is a "cluster", **NTP/Chrony needs to be installed and it's service running** - on all servers.
* System timezone should be correctly configured - on all servers.
* Install `fontconfig` and related OS packages - on all confluence servers. This will save you from some pain later.
* Create `confluence` user and `confluence` group - on all confluence servers - **with same UID and GID**.
* Synchronize necessary SSH keys to the DB and Confluence servers
* Improve NFS server (and client) performance
* Mount NFS share on all confluence servers on `/confluence`, with correct ownership and permissions
* **rsync** attachments from old server to `/confluence/shared-home/attachments` *before the actual migration*. This will save time later. Later, we will do the `rsync` again, which will copy the final changes (deltas) to the NFS mount-point.
* Install Postgres 9.6 on the new DB server
* Optimize Postgres memory parameters so it uses RAM optimally
* Configure memory (java heap) correctly on all confluence servers


## In-flight steps:
* Stop Confluence service on the production server, so the Confluence service does not change any data in the database, nor it changes any data files in its data directory.
* Configure the SSL load-balancer with session affinity and WebSockets support in front of the cluster, and make sure that the correct URL (depending on test or production migration) now points to the three new confluence nodes. Yes. This would need to be done right after we stop confluence service on the old server. The new setup would need to be accessed by the correct URL, so it can be configured correctly.
* Dump Confluence database from existing db server.
* Create Confluence database on new DB server, with correct encoding and correct collation
* Restore Confluence database dump on the new DB server.
* Install Confluence `6.4.0` on new confluence node-1 only
* Copy Confluence related data files from old server to new server, adjust configuration files, and start Confluence
* Perform basic checks
* Convert the confluence node-1 to data-center mode by applying "data-center license".
* Adjust files and re-locate `shared-home` and `attachments` directories.
* Upgrade version of Confluence on data-center node-1 from `6.4.0` to `7.4.3 LTS` 
* Perform basic checks (including Cluster checks)

## Post-touchdown checks:
* Update / fix plugins
* Add more nodes - one at a time
* Perform cluster related checks
* If everything works as expected, completely disable Confluence service on the old Confluence server.
* Done.

The actual steps are documented in the guide I mentioned earlier, available through [our github repository](https://github.com/Praqma/atlassian-guides/blob/master/confluence/confluence-6.4.0-standalone-to-7.4.3-datacenter.md).
