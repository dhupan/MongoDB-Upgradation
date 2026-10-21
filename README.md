# MongoDB Upgrade Lab: 3.4 / 3.6 to 8.3

A practical lab runbook consolidating two procedure documents for staged MongoDB upgrades on Linux. It covers environment preparation, binary extraction, database startup and shutdown, Feature Compatibility Version (FCV) checks, database statistics capture, and backup/restore examples.

> **Important — review before use:** This README is based on lab notes and is not a certified production runbook. The source documents contain version-label inconsistencies, commands that need validation, and an FCV instruction for 8.3 that must not be assumed valid. Verify the exact supported upgrade path and FCV requirements in the official documentation for every target release. Test the complete procedure on a restored copy before any production change.

## Scope

The supplied procedures describe a standalone MongoDB lab using databases named `speedy_search` and `log_next`. The documents include a path beginning at MongoDB 3.4, moving through 3.6, 4.0, 4.2, 4.4, 5.0, 6.0, 7.0, 8.0, and then an 8.3 binary.

The documents do **not** establish that every listed transition is supported for every operating system, patch release, data format, or deployment. Treat the sequence below as a record of the lab procedure, not a substitute for the official upgrade matrix.

## Lab configuration captured in the source notes

| Setting | Value in the procedure |
|---|---|
| Operating environment | Linux / Ubuntu-based binaries |
| Example database | `speedy_search` or `log_next` |
| Example data directory | `/var/lib/mongo34` or `/data/lib/log_next` |
| Example port | `27017` or `27018` |
| Bind address | `127.0.0.1` |
| Logs | Version-specific paths under `/var/log/` |
| Upgrade style | Standalone instance, one binary version at a time |

The two source documents use different database paths, ports, log paths, and database names. Choose one consistent set of values for your lab and verify every command before running it. Do not mix examples from the two environments without adapting them.

## Repository layout

```text
mongodb-upgradation/
├── README.md
├── docs/
│   ├── upgrade-checklist.md
│   ├── compatibility-notes.md
│   └── rollback-plan.md
├── procedures/
│   ├── standalone-upgrade.md
│   └── backup-and-restore.md
├── scripts/
│   └── database-statistics.js
└── evidence/
    └── sample-output/
```

## 1. Prepare the Linux environment

The source procedure installs basic dependencies, the MongoDB shell, and MongoDB Database Tools. Package names and versions should be chosen to match the target operating system and the MongoDB release being used.

Example dependency preparation from the source notes:

```bash
sudo apt update
sudo apt install libcurl4
sudo apt-get install -f
```

The source notes also download MongoDB Shell and Database Tools packages. Confirm their authenticity, compatibility, and supported download URLs before installation.

Create the data and log directories required by your chosen configuration. Example:

```bash
sudo mkdir -p /var/lib/mongo34
sudo mkdir -p /var/log/mongodb34
```

Ensure the account running `mongod` has the correct permissions. Do not run a database process against a data directory that is currently being used by another `mongod` process.

## 2. Extract the MongoDB binaries

The source documents use version-specific tar archives extracted under `/root`, including examples for MongoDB 3.4, 3.6, 4.0, 4.2, 4.4, 5.0, 6.0, 7.0, 8.0, and 8.3.

Example pattern:

```bash
cd /root
tar -xzf /path/to/mongodb-version-linux-x86_64.tgz
```

Use the archive that matches the operating system and version you have verified. Do not assume an Ubuntu 18.04 or Ubuntu 20.04 build is suitable for a different operating system.

## 3. Start MongoDB and connect

Example command pattern from the source notes:

```bash
/path/to/mongodb-version/bin/mongod \
  --dbpath /var/lib/mongo34 \
  --logpath /var/log/mongodb34/mongod.log \
  --bind_ip 127.0.0.1 \
  --port 27017 \
  --fork
```

Replace the binary, data path, and log path for each stage. Confirm the process is stopped cleanly before starting a different version against the same data directory.

Connect using a client compatible with the server version:

```bash
mongosh --host 127.0.0.1 --port 27017
```

Older releases may require the legacy `mongo` shell. Check the version and client compatibility rather than assuming `mongosh` works with every legacy server.

Check the server version:

```javascript
db.version()
```

Select the database used by your lab:

```javascript
show dbs
use speedy_search
```

Or, if your lab uses the other source database:

```javascript
use log_next
```

## 4. Record database statistics at every stage

Capture statistics before starting the upgrade and after every successful stage. Save each result in a separate text file named with the server version and date/time.

The following script is adapted from the supplied notes:

```javascript
var d = db.stats();

print("\nDatabase: " + db.getName());
print("Collections : " + d.collections);
print("Objects     : " + d.objects);
print("Data Size   : " + (d.dataSize / 1024 / 1024).toFixed(2) + " MB");
print("Storage Size: " + (d.storageSize / 1024 / 1024).toFixed(2) + " MB");
print("Index Size  : " + (d.indexSize / 1024 / 1024).toFixed(2) + " MB");

print("\nCollection Details:");

db.getCollectionNames().forEach(function (c) {
  var s = db.getCollection(c).stats();

  print(
    c +
    " | Docs: " + s.count +
    " | Size: " + (s.size / 1024 / 1024).toFixed(2) + " MB" +
    " | Storage: " + (s.storageSize / 1024 / 1024).toFixed(2) + " MB"
  );
});
```

### What to compare

- Database and collection counts
- Document counts per collection
- Data size, storage size, and index size
- Application-level checks and representative queries
- Server logs and error messages

Storage and index sizes can change between versions. A size difference alone does not prove data loss or corruption. Investigate unexpected document-count differences and validate critical application data.

## 5. Check and manage Feature Compatibility Version (FCV)

Before changing FCV, confirm the current value and read the official upgrade instructions for the exact source and target versions.

```javascript
use admin

db.adminCommand({
  getParameter: 1,
  featureCompatibilityVersion: 1
})
```

Only set FCV when the official upgrade procedure says it is appropriate for the target binary and the current upgrade stage. Example syntax used in the source notes for legacy transitions:

```javascript
db.adminCommand({
  setFeatureCompatibilityVersion: "TARGET_VERSION"
})
```

Replace `TARGET_VERSION` only with a value explicitly supported by the official instructions for that release. Some releases require additional command options or confirmation fields; use the version-specific documentation.

**Do not copy the source notes' `setFeatureCompatibilityVersion: "8.3"` command without verification.** Confirm the FCV values supported by the exact MongoDB 8.3 release and the official upgrade guide. Also re-check FCV after the command completes.

## 6. Staged version sequence recorded in the documents

The source notes describe the following sequence:

1. MongoDB 3.4 to 3.6
2. MongoDB 3.6 to 4.0
3. MongoDB 4.0 to 4.2
4. MongoDB 4.2 to 4.4
5. MongoDB 4.4 to 5.0
6. MongoDB 5.0 to 6.0
7. MongoDB 6.0 to 7.0
8. MongoDB 7.0 to 8.0
9. A further step from an 8.0 binary to an 8.3 binary

For each stage:

1. Review the official upgrade instructions and compatibility requirements.
2. Take and verify a backup.
3. Stop the running `mongod` cleanly.
4. Start the intended target binary with the correct data and log paths.
5. Check the server version and logs.
6. Check FCV and update it only when the official instructions require it.
7. Run database statistics and application validation.
8. Save the results and proceed only after the checks pass.

### Some technical buzzword  to resolve

from version 5.0 to 8.3 mongodb will be run by mongosh instead mongo
so need to use /usr/local/bin/mongosh instead /root/mongodb-4.6/bin/mongod \

## 7. Shut down cleanly between stages

Connect to the server and run:

```javascript
use admin
db.shutdownServer()
```

Verify that the `mongod` process has stopped before starting another version. Avoid routine forced termination because it may interrupt database shutdown and complicate recovery.

## 8. Backup and restore examples

The source documents use `mongodump` and `mongorestore` with gzip-compressed archives.

### Create a compressed database archive

```bash
mongodump \
  --host 127.0.0.1 \
  --port 27017 \
  --db speedy_search \
  --archive="/backup/speedy_search_$(date +%F_%H-%M-%S).gz" \
  --gzip
```

### Restore a compressed archive

```bash
mongorestore \
  --host 127.0.0.1 \
  --port 27017 \
  --archive=/backup/speedy_search_backup.gz \
  --gzip
```

When authentication is enabled, provide credentials using an approved secure method. Do not hard-code passwords in scripts, commit them to Git, or expose them in screenshots or logs. The uploaded source documents contained literal passwords; they have intentionally been omitted from this public-facing page.

Before restoring, confirm the destination host, port, database mapping, and whether existing collections may be affected. Test the backup by restoring it to an isolated environment and validating the resulting data.

## 9. Upgrade validation checklist

- [ ] Source version, target version, and supported path confirmed
- [ ] Operating system and binary compatibility confirmed
- [ ] Backup completed and restore tested
- [ ] Baseline database statistics saved
- [ ] Application and driver compatibility reviewed
- [ ] Correct `mongod` binary and data directory selected
- [ ] Server starts successfully after each stage
- [ ] Logs reviewed for errors and warnings
- [ ] FCV checked and handled according to the official instructions
- [ ] Collection and document counts compared
- [ ] Critical queries and application workflows tested
- [ ] Post-upgrade backup completed
- [ ] Recovery plan and evidence documented

## 10. Official documentation

Always use the documentation for the exact target release:

- [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [MongoDB Release Notes](https://www.mongodb.com/docs/manual/release-notes/)
- [MongoDB Upgrade Procedures](https://www.mongodb.com/docs/manual/release-notes/8.3-upgrade/)
- [Feature Compatibility Version command](https://www.mongodb.com/docs/manual/reference/command/setFeatureCompatibilityVersion/)
- [MongoDB Database Tools](https://www.mongodb.com/docs/database-tools/)

## Disclaimer

This repository documents a lab procedure consolidated from working notes. It is provided for learning and knowledge sharing, not as a guarantee that the sequence or every command is valid for all deployments. Validate the supported upgrade path, FCV, operating system, drivers, backup and restore strategy, and rollback plan before use in production.
