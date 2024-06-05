# MongoDB 7.0.11 EE Single Instance Architecture and Deployment in Redhat Linux 9

![MongoDB Architecture](https://github.com/MdAhosanHabib/MongoDB-Single-Instance-Architecture-and-Deployment/assets/43145662/e50ae2fc-bad5-4d90-b802-8adf680a619f)

An overview of MongoDB's architecture, particularly focusing on the WiredTiger storage engine and its components. Let's break down each part of the architecture and explain their roles and Deployment in Redhat Linux 9:

## 1. MongoDB Driver / App
Role: This represents the application or client that interacts with MongoDB. It sends queries and commands to the MongoDB server.


## 2. Query Engine
Components:

Command Parser: Parses commands and queries sent by the client.

Query Planner: Determines the most efficient way to execute a query.

DML (Data Manipulation Language): Handles operations like insert, update, and delete.

Role: The query engine processes and optimizes queries before they are executed by the storage engine.


## 3. Security
Role: Manages authentication and authorization, ensuring that only authorized users can access and manipulate the data.


## 4. Management
Role: Involves monitoring, backup, and other administrative tasks.


## 5. Storage Engine: WiredTiger
The WiredTiger storage engine is responsible for how data is stored, managed, and retrieved on disk.

#### Schema & Cursor
Role: Manages the structure of the data (schema) and provides a way to navigate through the data (cursors).

#### Transactions
Role: Ensures ACID properties (Atomicity, Consistency, Isolation, Durability) for multi-document operations, allowing for complex, reliable transactions.

#### Row Storage & Column Storage
Role:

Row Storage: Stores data in a row-based format.

Column Storage: Stores data in a columnar format, optimized for specific types of queries.

#### Snapshots
Role: Creates consistent snapshots of the data at a given point in time, useful for backups and replication.

#### Cache
Role: Stores frequently accessed data in memory to speed up read and write operations.

#### Page Read/Write
Role: Manages the reading and writing of data pages between memory and disk.

#### Block Management
Role: Handles the allocation and management of disk space for storing data.

#### WAL (Write-Ahead Logging)
Role: Logs all changes before they are applied to the database. This helps in data recovery in case of a crash.

## 6. DB Files
Role: Physical files on disk where the actual data is stored.


## 7. Journal
Role: A separate file that logs write operations to ensure data integrity and support crash recovery.


## Oplog (Operations Log)
Role:

Location: The oplog is typically stored in the local database under the collection oplog.rs.

Function: The oplog is a capped collection that logs all write operations on the primary node. It is used for replication to ensure that secondary nodes can apply the same operations in the same order, maintaining data consistency across replicas.

Management: The size of the oplog is configurable, and it can be managed to ensure it doesn't consume excessive disk space. Older entries are automatically purged once the oplog reaches its size limit.


## Checkpointing
Role: Periodically writes the in-memory state of the database to disk, ensuring a consistent state and reducing the amount of data that needs to be replayed from the journal during recovery.


# Now step down the deployment in Redhat Linux 9 Environment

## Install MongoDB 7.0.11
```bash
-- go to this dir
cd /home/test


-- install the dependencies
dnf install cyrus-sasl -y
dnf install cyrus-sasl-gssapi -y
dnf install cyrus-sasl-plain -y


-- upload & install the required packages
rpm -ivh mongodb-database-tools-rhel90-x86_64-100.9.4.rpm
rpm -ivh mongodb-enterprise-server-7.0.11-1.el9.x86_64.rpm
rpm -ivh mongodb-mongosh-2.2.6.x86_64.rpm


-- create the Configuration file
vi /etc/mongod.conf

# Where and how to store data.
storage:
  dbPath: /test/mongodb7011
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # Half of the system memory
  directoryPerDB: true  # This option was missed in the original configuration
  oplogMinRetentionHours: 48 # Minimum retention period for the oplog in hours

# Where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /test/mongodb7011/mongod.log

# Network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0  # Bind to all interfaces

# Process Management
processManagement:
  fork: true  # Fork server process
  pidFilePath: /test/mongodb7011/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# Operation Profiling
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

# Replication
replication:
  replSetName: rs0
  oplogSizeMB: 1024  # 1GB oplog size


-- create & permission for data & archive
mkdir -p /test/mongodb7011
chown -R mongod:mongod /data/mongodb7011 /etc/mongod.conf


-- start mongodb & log
systemctl start mongod
systemctl enable mongod
systemctl status mongod

tail -1000f /test/mongodb7011/mongod.log


-- set the oplog
mongosh
rs.initiate()
rs.status()
use local
db.oplog.rs.stats()
```

## MongoDB User, Database and Collection Create
```bash
-- create super users & give permission
mongosh
Enterprise test> use admin
Enterprise admin> db.createUser({
  user: "admin",
  pwd: passwordPrompt(), // This will prompt you to enter a password securely
  roles: ["root"]
})

User: admin
pass: test


-- create database and Collection
use testdb
db.createCollection("exampleCollection")

db.exampleCollection.insertMany([
  {
    name: "Ahosan",
    age: 28,
    email: "ahosan@example.com",
    address: {
      street: "456 Oak St",
      city: "Othertown",
      state: "TX",
      zip: "67890"
    },
    hobbies: ["cooking", "biking"]
  },
  {
    name: "Habib",
    age: 27,
    email: "habib@example.com",
    address: {
      street: "789 Pine St",
      city: "Sometown",
      state: "NY",
      zip: "11223"
    },
    hobbies: ["gaming", "hiking"]
  }
])

Enterprise admin> db.createUser({
  user: "test",
  pwd: passwordPrompt(), // This will prompt you to enter a password securely
  roles: [{ role: "readWrite", db: "testdb" }] // Replace "testdb" with the name of your database
})

User: test
Pass: test


-- add in conf of mongod
openssl rand -base64 756 > /test/mongodb7011/keyfile
chown -R mongod:mongod /test/mongodb7011/keyfile
chmod 400 /test/mongodb7011/keyfile

vi /etc/mongod.conf
# Security settings
security:
  authorization: enabled
  keyFile: /test/mongodb7011/keyfile

systemctl restart mongod


-- login by app user
mongosh -u test -p --authenticationDatabase testdb
Pass: test
use testdb
db.exampleCollection.find({ age: 27 })


-- login to super user
mongosh -u admin -p --authenticationDatabase admin
Pass: test
```

## MongoDB credentials for Applications or MongoDBCompass
```bash
-- super user
IP: 192.0.0.5
Port: 27017
User: admin
Pass: test
authenticationDatabase: admin

-- app user
IP: 192.0.0.5
Port: 27017
User: test
Pass: test
authenticationDatabase: testdb
```

That's all.

Regards from Ahosan.

Medium: 
