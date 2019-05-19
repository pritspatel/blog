---
title: "Leadership election pattern"
date: 2019-05-19
tags: [spring integration, leadership, election, jdbc, java, spring boot]
---

### Leadership Election Pattern

#### Problem

Sometime we want to run multiple instance of our spring integration application but want only
one instance to poll some common resource like db or file system etc to build fault tolerant system
.So in case one instance goes down then another will take over the work.

So how can you implement such pattern using spring integration and shared db table?

#### Solution

In this blog post I will share my ideas about how I solved above mentioned problem. Recently I was working on a project
where we were using master-slave design pattern where master node of the spring integration application will poll some
common resource (eg. db in our case) and delgate that work to multiple worker node for horizontal scalability. What I 
expereinced is that when my master node went down, whole processing pipeline went down.
 
After doing some research, I found out that we need to run multiple instance of Master node and allow only one node at a
time to poll and distribute work. In case active master node crash, then another passive master node will become active.
To implement this pattern, I found out that spring integration has built in mechanism. Luckily we were using spring ecosystem
in our project so that made our job easy.

So how does all this work together. 

1. Add following dependencies in your pom.xml

```xml

        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-integration</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.integration</groupId>
			<artifactId>spring-integration-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
```

2. Create following table. You can find the script for this table from spring integration github repo for your respective db. This example uses mysql. [spring-integration-jdbc-script]('https://github.com/spring-projects/spring-integration/tree/master/spring-integration-jdbc/src/main/resources/org/springframework/integration/jdbc')

```sql

CREATE TABLE INT_LOCK  (
	LOCK_KEY CHAR(36) NOT NULL,
	REGION VARCHAR(100) NOT NULL,
	CLIENT_ID CHAR(36),
	CREATED_DATE DATETIME(6) NOT NULL,
	constraint LOCK_PK primary key (LOCK_KEY, REGION)
) ENGINE=InnoDB;
```

3. Add following java configuration

```java

    @Bean
    public DefaultLockRepository lockRepository(DataSource dataSource) {
        return new DefaultLockRepository(dataSource);
    }

    @Bean
    public JdbcLockRegistry lockRegistry(LockRepository lockRepository) {
        return new JdbcLockRegistry(lockRepository);
    }

    @Bean
    public LockRegistryLeaderInitiator leaderInitiator(LockRegistry lockRegistry) {
        return new LockRegistryLeaderInitiator(lockRegistry, leaderCandidate());
    }
```

4. You will also need LeaderCadidate class which must extend org.springframework.integration.leader.DefaultCandidate class, so when spring integration elect the leader, it will notify
this leader candidate class. It will also notify this class when leadership is revoked. So you can implement certain business logic on those events. For the purpose of this demo, we will
start/stop the spring integration jdbc poller

```java

@Component
public class LeaderCandidate extends DefaultCandidate {

    @Autowired
    @Qualifier("systemMsgChannel")
    private MessageChannel systemMessageChannel;

    @Override
    public void onGranted(Context context) {
        super.onGranted(context);
        System.out.println("*** Leadership granted ***");
        System.out.println("STARTING JDBC POLLER");
        Message<String> startMsg = MessageBuilder.withPayload("@jdbcPoller.start()").build();
        systemMessageChannel.send(startMsg);
        System.out.println("STARTUP MESSAGE SENT");

    }

    @Override
    public void onRevoked(Context context) {
        super.onRevoked(context);
        System.out.println("*** Leadership revoked ***");
        System.out.println("STOPPING JDBC POLLER");
        Message<String> stringMessage = MessageBuilder.withPayload("@jdbcPoller.stop()").build();
        systemMessageChannel.send(stringMessage);
        System.out.println("SHUTDOWN MESSAGE SENT");
    }
}

```

As you can see from different code snippets, when application starts ```LockRegistryLeaderInitiator``` class will try to acquire
lock. Since multiple instance of the same app is running, only one will be able to acquire lock against the table INT_LOCK. If it
is successfull then spring integration will fire event which will be listened by ```LeaderCandidate``` class ```onGranted``` method.
Here now you can start your jdbc poller or anyother custom biz logic you can implement. Other instances will not be granted leadership
and will stay on standby mode. Now if you kill the instance which was selected as leader, you will notice that, other instance will be
granted leadership and will take over the work.


Complete source code of this project can be found on [leader-election-demo]('https://github.com/pritspatel/leader-election-demo').

Please feel free to comment or suggest better idea. Goal of this post is to share ideas and learn from others.