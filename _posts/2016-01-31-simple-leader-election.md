---
layout: post
title: "Simple Leader Election"
description: "Simple leader election based on pessimistic locks implemented on shared MongoDB database with C#"
category: 
tags: [Leader Election, Pessimistic Lock, MongoDB]
keywords: [leader election pessimistic lock mongodb destributed computing c#]
published: true
---
{% include JB/setup %}

#Overview:
From times to times it is required to run specific job only by one instance of an application, a service, etc. For example if you scale you system horizontally and would like to run specific job only on one node (server) or you are planning to roll out new version of the system with zero downtime and would like to prevent specific job to be executed by several processes at the same time. In this post I will describe how to achieve such behaviour on C# by using pessimistic locks on shared MongoDB database. The same approach could be implemented on others kind of DBMS as well.
<!--more-->

#Requirements:
1.	To run job only by one instance (Leader) of system unit (Application, Service and so on).
2.	In case of leader became unavailable one of the others live instances must take leadership and continue job processing.  

#Solution:

Requirements could be fulfilled with pessimistic locks. Each instance must acquire a lock with well-known id before executing unit of work which should be run only by one instance at a time. If lock is acquired by one instance then it is allowed to execute unit of work by that instance. Once instance will be turned off it should release acquired lock in order to let left instances to continue job processing. In case of leader became unavailable without releasing lock it should be released automatically after timeout. In order to keep lock acquired leader instance should update acquired lock periodically.

#MongoDB Pessimistic Lock implementation:
Let’s define a dedicated MongoDB collection *PessimisticLocks*.

{% highlight csharp %}
locks = database.GetCollection<PessimisticLockDto>("PessimisticLocks");
{% endhighlight %}

It will contain documents with the following structure:
{% highlight csharp %}
internal class PessimisticLockDto
{
	public string Id { get; set; } 
	[BsonElement("ts")]
	public DateTime TimeStamp { get; set; } //Timestamp of the lock
	[BsonElement("kid")]
	public string KeeperId { get; set; } //lock Keeper Id
}
{% endhighlight %}
We need to create the following indexes on this collection.

<ol><li>
Index on <code>TimeStamp</code> field with time to live e.g. 90 seconds. This will allow MongoDB to remove documents automatically when lock timeout will be reached. This feature is supported by MongoDB since version 2.2. For more info look 
<a href="https://docs.mongodb.org/manual/tutorial/expire-data/">here</a>
{% highlight csharp %}
locks.CreateIndex(
	new IndexKeysBuilder()
		.Ascending("ts"), 
	new IndexOptionsBuilder()
		.SetTimeToLive(TimeSpan.FromSeconds(ttl))
		.SetName("ts"));
{% endhighlight %}
</li><li>
Index on <code>KeeperId</code> field. It will allow to release locks by keeper id a little bit quicker.
{% highlight csharp %}
locks.CreateIndex(
	new IndexKeysBuilder().Ascending("kid"), 
	new IndexOptionsBuilder().SetUnique(false));
{% endhighlight %}
</li></ol>

##Acquire Lock

In order to acquire lock we need insert document with well-known id, current time stamp and unique instance id (`KeeperId`). As id should be unique in the collection only first instance will be able to insert document (acquire lock). All others instances will receive duplicate key exception. In order to keep lock acquired we need to update `TimeStamp` field with current date periodically. E.g. every 30 seconds. This interval should be less than TTL value on `TimeStamp` index in other case lock will be released by MongoDB earlier than required. Based on this `AcquireLock` method could look as following.
{% highlight csharp %}
public bool AcquireLock(string lockId, string keeperId)
{
	try
	{
		return locks.Update(Query<PessimisticLockDto>
			.Where(x => x.Id == lockId && x.KeeperId == keeperId),
			Update<PessimisticLockDto>.CurrentDate(x => x.TimeStamp), 
			UpdateFlags.Upsert, 
			WriteConcern.WMajority).DocumentsAffected == 1;
	}
	catch (Exception)
	{
		return false;
	}
}
{% endhighlight %}

It is worth to add that insert/update operation should be executed with Write Concern *Majority* as in other case [MongoDB Data Rollback](https://docs.mongodb.org/manual/core/replica-set-rollbacks/) could cause several instances to become a leader at the same time.

##Release Lock
<ol>
<li>
Lock could be released manually by removing document by <code>Id</code> and <code>KeeperId</code>. It is important to include <code>KepperId</code> in remove query as well as <code>Id</code> as in other case it is possible to release locks acquired by one of the others processes. 
Based on those <code>ReleaseLock</code> method could look as following.
{% highlight csharp %}
public void ReleaseLock(string lockId, string keeperId)
{
	locks.Remove(Query<PessimisticLockDto>
		.Where(x => x.Id == lockId && x.KeeperId == keeperId), 
		WriteConcern.WMajority);
}
{% endhighlight %}
</li><li>
Lock could be released by MongoDB background process which will compare <code>TimeStamp</code> of lock document with TTL value in index and remove document if it is expired. As result lock will be released and one of the left instances will be able to acquire lock and continue processing.
</li>
</ol>

Once we have Pessimistic lock implemented we could implement Leader Aware Task Runner.

#Leader Aware Task Runner implementation:

In order to use pessimistic lock we need to choose keeper id and lock id.
As `KepperId` we could generate the following string.
{% highlight csharp %}
public static string GenerateLockKeeperId()
{
	var process = Process.GetCurrentProcess();
	return string.Format("{2}-LeaderAwareTaskRunner-PID-{0}-Domain-{1}",
	process.Id, AppDomain.CurrentDomain.FriendlyName, process.MachineName);
}
{% endhighlight %}
Such id will allow to run several instances on separate nodes as well as on the same node in different processes or even in one process but separate application domains.
As lock id we need to select a string which uniquely describe job across all instances. e.g. unique job name.

Method which elects a leader could look like the following
{% highlight csharp %}   
private bool IsStillALeader()
{
	// try to acuire lock if not a leader or after 30 seconds of became a leader
	if (!isALeader || watch.ElapsedMilliseconds > lockPingTimeoutInSeconds * 1000) 
	{
		isALeader = locker.AcquireLock(lockId, lockKeeperId);
		if (isALeader) watch.Restart();
	}
	return isALeader;
}
{% endhighlight %}
If current process is not a leader it will try to acquire lock in order to became a leader. Or if lock ping timeout has been reached it will try to acquire lock once more in order to update time stamp on lock document and keep lock as acquired.
 
And method which run a task could look as following
{% highlight csharp %}
public void RunTaskOnLeader(Action task)
{
	if (IsStillALeader())
		task();
}
{% endhighlight %}
Now we are able to run task in a loop or by timer on several instances and task will be executed only on one of them. The only requirement here is that task should be finished in less than TTL value of timestamp index. If task runs for longer period of time it is required either to increase a TTL value for index or when lock is acquired run a timer which will keep lock acquired by making a call to acquire lock for example each 30 seconds if TTL on index greater than 30 seconds e.g. 90 seconds.

#Summary: 
Proposed solution forbid to run a task on more than one instance at a time. In the same time if leader will become unavailable the task will start to run on one of the left instances.

Source code with examples could be find [here](https://github.com/vedmiediev/SimpleLeaderElection)

And thanks for reading so far.				
