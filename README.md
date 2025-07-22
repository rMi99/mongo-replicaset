
````markdown
# 🗃️ MongoDB Replica Set Setup (Test Environment)

This guide walks you through setting up a **MongoDB replica set** with an arbiter, using test domains for a QA or staging environment.

---

## 🔧 Step 1: Add Host Entries

Add the following to the `/etc/hosts` file **on all MongoDB nodes**:

```bash
192.168.1.114 mongo-node-1.test-mongo.qa.myapp.io
192.168.1.147 mongo-node-2.test-mongo.qa.myapp.io
192.168.1.220 mongo-node-3.test-mongo.qa.myapp.io
192.168.1.128 mongo-arbiter.test-mongo.qa.myapp.io
127.0.0.1 mongo-node-3.test-mongo.qa.myapp.io
````

---

## 🛠️ Step 2: Update `mongod.conf`

On each node, edit:

```bash
sudo nano /etc/mongod.conf
```

Ensure the following configuration exists:

```yaml
net:
  bindIp: 0.0.0.0
  port: 27017

replication:
  replSetName: rs0

security:
  authorization: enabled
```

Restart MongoDB:

```bash
sudo systemctl restart mongod
```

---

## 🔐 Step 3: Create Admin User (on Primary)

On `mongo-node-1`:

```bash
mongosh --host localhost
```

Then:

```js
use admin

db.createUser({
  user: "admin",
  pwd: "PASSWORD",
  roles: [ { role: "root", db: "admin" } ]
})
```

Reconnect with authentication:

```bash
mongosh --host localhost -u admin -p PASSWORD --authenticationDatabase admin
```

---

## 🧬 Step 4: Initiate Replica Set

Still on `mongo-node-1`, run:

```js
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-node-1.test-mongo.qa.myapp.io:27017" },
    { _id: 1, host: "mongo-node-2.test-mongo.qa.myapp.io:27017" },
    { _id: 2, host: "mongo-node-3.test-mongo.qa.myapp.io:27017" },
    { _id: 3, host: "mongo-arbiter.test-mongo.qa.myapp.io:27017", arbiterOnly: true }
  ]
})
```

You can later reconfigure with:

```js
rs.reconfig({...})
```

---

## ✅ Step 5: Validate Replica Set

Run:

```js
rs.status()
```

You should see:

* One node as `PRIMARY`
* Two nodes as `SECONDARY`
* One node as `ARBITER`

---

## 🔍 Step 6: Connect via URI (Secondary Read)

Use this connection URI:

```
mongodb://admin:PASSWORD@mongo-node-1.test-mongo.qa.myapp.io:27017,mongo-node-2.test-mongo.qa.myapp.io:27017,mongo-node-3.test-mongo.qa.myapp.io:27017/?replicaSet=rs0&readPreference=secondaryPreferred&authSource=admin
```

Test with `mongosh`:

```bash
mongosh "mongodb://admin:PASSWORD@mongo-node-1.test-mongo.qa.myapp.io:27017,mongo-node-2.test-mongo.qa.myapp.io:27017,mongo-node-3.test-mongo.qa.myapp.io:27017/?replicaSet=rs0&readPreference=secondaryPreferred&authSource=admin"
```

---

## 🧪 Optional: Verify Read from Secondary

Run:

```js
db.runCommand({ isMaster: 1 })
```

You’ll see `"secondary": true` if connected to a secondary.

---

## 👤 Need a Read-Only User?

> You can create a read-only user with:

```js
use admin

db.createUser({
  user: "readonly",
  pwd: "READONLY_PASS",
  roles: [ { role: "read", db: "your_database_name" } ]
})
```

---

## 🧼 Security Reminder

Never use real production credentials or public IPs in test guides pushed to GitHub. Always mask sensitive data when publishing!

---
