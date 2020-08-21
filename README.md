```bash
# bring up two postgre instances
docker-compose up -d
# cp config to overwrite the master config
cp backup/postgresql.conf backup/pg_hba.conf collector/
# master
docker-compose exec -u postgres collector-test bash
cat /mnt/backup/global.sql | psql
createdb repmgr -O repmgr

psql
ALTER USER repmgr SET search_path TO repmgr, "$user", public;

# exit and restart the container
docker-compose restart collector-test
docker-compose exec -u postgres collector-test bash
repmgr -f /mnt/backup/collector_master.conf primary register

# exit and enter slave
docker-compose exec collector-test-backup bash
chown postgres -R /var/lib/postgresql/data
su postgres
repmgr -h collector-test -p 5432 -U repmgr -d repmgr -f /mnt/backup/collector_standby.conf standby clone --dry-run
repmgr -F -h collector-test -p 5432 -U repmgr -d repmgr -f /mnt/backup/collector_standby.conf standby clone

# exit and restart the container
docker-compose restart collector-test-backup
docker-compose exec -u postgres collector-test-backup bash
pg_ctl -D /var/lib/postgresql/data start
repmgr -f /mnt/backup/collector_standby.conf standby register
```
```sql
-- now run this on master
docker-compose exec -u postgres collector-test bash
psql
CREATE TABLE entries1 (
  title text,
  body text
);
INSERT INTO entries1 VALUES ('PGroonga', 'PGroonga is a PostgreSQL extension for fast full text search that supports all languages. It will help us.');
INSERT INTO entries1 VALUES ('Groonga', 'Groonga is a full text search engine used by PGroonga. We did not know about it.');
INSERT INTO entries1 VALUES ('PGroonga and replication', 'PGroonga 1.1.6 supports WAL based streaming replication. We should try it!');
CREATE EXTENSION pgroonga;
CREATE INDEX entries1_full_text_search ON entries1 USING pgroonga (title, body);
SELECT pgroonga_command('io_flush');
SELECT title FROM entries1 WHERE title %% 'replication';

-- and run this on slave(fails)
docker-compose exec -u postgres collector-test-backup bash
psql
SELECT title FROM entries1 WHERE title %% 'replication';
```
