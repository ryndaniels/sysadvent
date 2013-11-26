## Upgrading Mongo for Fun and Profit

So you're running MongoDB and you want to upgrade it? Maybe you want some of those nifty [hashed shard keys](http://docs.mongodb.org/manual/core/sharding-shard-key/#sharding-hashed-sharding) in 2.4? This post will discuss my experiences upgrading a sharded replica set in production.

### Preparation

To begin, make sure you're familiar with the [official MongoDB upgrade notes](http://docs.mongodb.org/manual/release-notes/2.4-upgrade/#upgrade-a-sharded-cluster-from-mongodb-2-2-to-mongodb-2-4). Specifically, pay attention to the warning section stating basically that you shouldn't do anything interesting with your databases during the upgrade process. Messing with sharding, creating or dropping collections, and creating or dropping databases are to be avoided if you value your data integrity.

At this point, get some coffee, and make any other preparations needed for your environment. This might include silencing any MMS/Sensu/Nagios/etc. alerts for the duration of the upgrade. In our environment, we have a service called Downtime Abbey that we enable during things like Mongo upgrades - it allows us to redirect traffic to a downtime status page instead of displaying 500 errors caused by `mongos` clients reconnecting for a more pleasant and reliable end user experience.

### Upgrading!

The upgrade process starts by replacing the existing mongo binaries with the shiny new ones. If you're using some config management tool like Puppet or Chef, this can be as simple as bumping the `mongodb[version]` attribute to your desired version (2.4.8 is recommended as of this writing) and triggering that to run across your infrastructure as needed. (And if you aren't using config management, [you really should be](http://sysadvent.blogspot.com/2011/12/day-19-why-use-configuration-management.html)!)

#### A Detour, Perhaps.


(make sure readahead is set properly)
(if applicable: disable balancer)
on one worker: mongos --configsvr=<list> --upgrade
restart config servers in reverse order
tail mongos logs to make sure all is not fucked
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

