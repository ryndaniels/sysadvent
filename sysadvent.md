## Upgrading Mongo for Fun and Profit

So you're running MongoDB and you want to upgrade it? Maybe you want some of those nifty [hashed shard keys](http://docs.mongodb.org/manual/core/sharding-shard-key/#sharding-hashed-sharding) in 2.4? This post will discuss my experiences upgrading a sharded replica set in production.

### Preparation

To begin, make sure you're familiar with the [official MongoDB upgrade notes](http://docs.mongodb.org/manual/release-notes/2.4-upgrade/#upgrade-a-sharded-cluster-from-mongodb-2-2-to-mongodb-2-4). Specifically, pay attention to the warning section stating basically that you shouldn't do anything interesting with your databases during the upgrade process. Messing with sharding, creating or dropping collections, and creating or dropping databases are to be avoided if you value your data integrity.

At this point, get some coffee, and make any other preparations needed for your environment. This might include silencing any MMS/Sensu/Nagios/etc. alerts for the duration of the upgrade. In our environment, we have a service called Downtime Abbey that we enable during things like Mongo upgrades - it allows us to redirect traffic to a downtime status page instead of displaying 500 errors caused by `mongos` clients reconnecting for a more pleasant and reliable end user experience.

### Upgrading!

The upgrade process starts by replacing the existing mongo binaries with the shiny new ones. If you're using some config management tool like Puppet or Chef, this can be as simple as bumping the `mongodb[version]` attribute to your desired version (2.4.8 is recommended as of this writing) and triggering that to run across your infrastructure as needed. (And if you aren't using config management, [you really should be](http://sysadvent.blogspot.com/2011/12/day-19-why-use-configuration-management.html)!)

#### A Detour, Perhaps.

Since upgrading requires restarting all your `mongod` instances anyways, it can be a good excuse to do any tuning or make other adjustments that require such a restart. In our case, we realized that our disk readahead settings weren't optimal (MongoDB, Inc. recommends that [EC2 users use a readahead value of 32](http://docs.mongodb.org/manual/administration/production-notes/#ec2)) so we deployed a fix for that in Chef before continuing with the process, to cut down on the number of `mongod` restarts needed.

Also, if you are running the balancer, now is a good time to stop it.
```mongos> sh.setBalancerState(false)
mongos> sh.getBalancerState()
false
mongos>```

If the balancer is currently running a migration, it will wait for that chunk to finish before stopping. You may have to wait a few minutes for `sh.getBalancerState()` to return false before continuing. Also note that disabling (and later enabling) the balancer should only be done using a `mongos` connection as this will ensure that the changes are made using a two-phase commit, making sure that the changes are properly replicated to all config servers. Making this kind of change to the config database directly can lead to DifferConfig errors which can leave your cluster metadata read-only at best and invalid at worst.

#### Upgrading the Config Servers

Upgrading from Mongo 2.2 to 2.4 requires that you upgrade the config database servers, in order to update the metadata for the existing cluster. Before proceeding, make sure that all members of your cluster are running 2.2, not 2.0 or earlier. Watch out for 2.0 `mongos` processes as these will get in the way of upgrades as well. If you find any (hiding on a long-forgotten server in a corner of the data center somewhere), you'll need to wait at least 5 minutes after they have been stopped to continue. Once your cluster is entirely on 2.2, start a `mongos` 2.4 instance with the upgrade flag:

```mongos --upgrade --configsvr=config1.example.com,config2.example.com,config3.example.com```

If that `mongos` starts without errors, you can restart the `mongod` processes on your config servers one at a time, in the opposite order than how they were listed in the above command. The order above should be the same order that the rest of your mongos processes are running. It's important to make sure each config database process has restarted completely before moving onto the next one to make sure the configuration data stays in sync across the 3 servers. You may want to `tail -f` the mongos logs during this process to keep an eye out for any errors.

### Restarting ALL the Things!
![Restart ALL the Mongos!](http://cdn.memegenerator.net/instances/500x/43326424.jpg)

Now it's time to restart all the `mongod` processes so they will start up using the fancy new 2.4 binaries, starting with the secondaries and arbiters.

#### Secondary Members and Arbiters

For each shard, if you aren't certain which members are primary or secondary, you can find out with the replica set status command:
```shard2:SECONDARY> rs.status()
{
    "set" : "shard2",
    "date" : ISODate("2013-11-26T19:31:49Z"),
    "myState" : 2,
    "syncingTo" : "shard2-db3.example.com",
    "members" : [
        {
            "_id" : 21,
            "name" : "shard2-db3.example.com",
            "health" : 1,
            "state" : 1,
            "stateStr" : "PRIMARY",
            "uptime" : 28196,
            "optime" : Timestamp(1385494307, 3),
            "optimeDate" : ISODate("2013-11-26T19:31:47Z"),
            "lastHeartbeat" : ISODate("2013-11-26T19:31:48Z"),
            "lastHeartbeatRecv" : ISODate("2013-11-26T19:31:48Z"),
            "pingMs" : 1
        },
        {
            "_id" : 26,
            "name" : "shard2-dr1.example.com",
            "health" : 1,
            "state" : 2,
            "stateStr" : "SECONDARY",
            "uptime" : 28196,
            "optime" : Timestamp(1385494307, 3),
            "optimeDate" : ISODate("2013-11-26T19:31:47Z"),
            "lastHeartbeat" : ISODate("2013-11-26T19:31:48Z"),
            "lastHeartbeatRecv" : ISODate("2013-11-26T19:31:48Z"),
            "pingMs" : 0,
            "syncingTo" : "shard2-db3.example.com"
        },
        {
            "_id" : 27,
            "name" : "shard2-db1.example.com",
            "health" : 1,
            "state" : 2,
            "stateStr" : "SECONDARY",
            "uptime" : 98979,
            "optime" : Timestamp(1385494309, 1),
            "optimeDate" : ISODate("2013-11-26T19:31:49Z"),
            "self" : true
        }
    ],
    "ok" : 1
}```

For each shard, the things to note are which replica set members are primary versus secondary, and if the replica set is currently "ok". If you are looking to avoid downtime, each shard and each member should be restarted one at a time; in our experience with a scheduled maintenance window we were able to restart all of our secondary members (we have a 3-member replica set across 4 shards, giving us 4 primary members and 8 secondaries) at once without issue. [Arbiters](http://docs.mongodb.org/manual/core/replica-set-arbiter/), if you have them, can be restarted at this time in any order.

#### Primary Members

Now we come to the most fun part of the upgrade- restarting the primaries! The details of completing this step will depend somewhat on how much (if any) downtime you can tolerate. Because Mongo's secret webscale sauce allows it to be relatively fault-tolerant, you can simply restart the primary `mongod` processes and allow the normal failover process to happen. However, this method takes longer than running `rs.stepDown()` on the primary nodes (which is MongoDB, Inc's suggested method) so if you aren't operating in a scheduled maintenance window, you will want to use the stepdown method to minimize downtime.

It should be noted that in version 2.2 and earlier, old `mongos` connections are only disabled after they have been tried (and failed). This can cause undesirable behavior such as intermittent errors showing up for end users as various `mongos` connections fail and get disabled. It is recommended to restart all the `mongos` connections before restarting the primary members of the replica set in order to avoid these issues. If you want to totally avoid downtime, these restarts must be done one at a time. If you have more than a few `mongos` connections in your infrastructure, it's best to do this upgrade during a scheduled maintenance window to restart them all at once and avoid that process.

Once you have finished restarting the primary replica set members, verify your success! The Mongo shell command `rs.status()` will show you which members are the new primaries as well as verifying that each shard has all replica set members and arbiters present and healthy, and running `mongo --version` will verify if you have, in fact, managed to upgrade to the version you wanted.

### Final Steps

Once you've verified that the upgrade has been successful, be sure to take care of any loose ends related to the maintenance, such as re-enabling alerts, ending any downtime windows, and re-enabling the balancer (if applicable). Assuming you were properly caffeinated, followed these steps, and paid attention to any errors Mongo may have helpfully provided, you should be the proud owner of a 2.4 sharded replica set. And if something went terribly terribly wrong, you did have [backups](http://docs.mongodb.org/manual/core/backups/), right?
