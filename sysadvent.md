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

If the balancer is currently running a migration, it will wait for that chunk to finish before stopping. You may have to wait a few minutes for `sh.getBalancerState()` to return false before continuing.

#### Upgrading the Config Servers

Upgrading from Mongo 2.2 to 2.4 requires that you upgrade the config database servers, in order to update the metadata for the existing cluser. Before proceeding, make sure that all members of your cluster are running 2.2, not 2.0 or earlier. Watch out for 2.0 `mongos` processes as these will get in the way of upgrades as well. If you find any (hiding on a long-forgotten server in a corner of the data center somewhere), you'll need to wait at least 5 minutes after they have been stopped to continue. Once your cluster is entirely on 2.2, start a `mongos` 2.4 instance with the upgrade flag:

```mongos --upgrade --configsvr=config1.example.com,config2.example.com,config3.example.com```

If that `mongos` starts without errors, you can restart the `mongod` processes on your config servers one at a time, in the opposite order than how they were listed in the above command. The order above should be the same order that the rest of your mongos processes are running. It's important to make sure each config database process has restarted completely before moving onto the next one to make sure the configuration data stays in sync across the 3 servers. You may want to `tail -f` the mongos logs during this process to keep an eye out for any errors.

#### Restarting ALL the Things!
![Restart ALL the Mongos!](http://cdn.memegenerator.net/instances/500x/43326424.jpg)

knife ssh service mongos restart
restart secondaries
restart arbiters
restart log dbs
restart primaries
reload site apps: workers, web, api, push
verify
how to verify when downtime abbey is on
(enable balancer)
turn off downtime abbey, etc
scotch

