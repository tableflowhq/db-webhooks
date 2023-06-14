<div align="center">
<h1 align="center">🪝 DB Webhooks</h1> </h1>
<em>Real-time events for Postgres</em>

</div>

DB Webhooks is a utility for Postgres that triggers webhooks when rows are inserted, updated, or deleted. It uses
database triggers that send low-latency websocket messages to a Go application. This application then calls
the configured webhook(s) with a JSON payload that includes specified values from the database row.

### How It Works

1. Data is modified in a Postgres table (INSERT, UPDATE, DELETE)
2. A Postgres trigger notifies the DB Webhooks web server via a websocket message
3. DB Webhooks formats, filters, and sends the data to configured webhook(s)

![DB Webhooks Create Slack Notification](https://i.imgur.com/1xoorz9.gif)

## Get Started

### Run DB Webhooks locally

You can run DB Webhooks locally with Docker.

```bash
git clone --depth 1 https://github.com/tableflowhq/db-webhooks.git
cd db-webhooks
docker-compose up -d
```

Then open [http://localhost:3000](http://localhost:3000) to access DB Webhooks.
<br>
<br>
**Note**: When connecting your database, if your Postgres host is `localhost`, you must use `host.docker.internal`
instead to access it when running with Docker.

### Run DB Webhooks on AWS (EC2)

**Note**: Make sure this instance is only accessible within your VPC.\
**Note**: Make sure your local machine is able to connect to the server on port 3000 (the web server) and 3003 (the API
server) over HTTP.\
**Note**: These instructions are for Amazon Linux 2 AMI (HVM).

#### Option 1 (one-line install)

```bash
sudo yum update -y && \
sudo yum install -y docker && \
sudo service docker start && \
sudo usermod -a -G docker $USER && \
sudo curl -L "https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
sudo mv /usr/local/bin/docker-compose /usr/bin/docker-compose && \
sudo chmod +x /usr/bin/docker-compose && \
mkdir db-webhooks && cd db-webhooks && \
wget https://raw.githubusercontent.com/tableflowhq/db-webhooks/main/{.env,docker-compose.yml,.dockerignore,frontend.env} && \
sg docker -c 'docker-compose up -d'

```

#### Option 2 (guided install)

1. To install Docker, run the following command in your SSH session on the instance terminal:

```bash
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker $USER
logout # Needed to close the SSH session so Docker does not have to be run as root
```

2. To install `docker-compose`, run the following command in your ssh session on the instance terminal:

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-$(uname -s)-$(uname -m)"  -o /usr/local/bin/docker-compose
sudo mv /usr/local/bin/docker-compose /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
docker-compose version
```

3. Install and run DB Webhooks

```bash
mkdir db-webhooks && cd db-webhooks
wget https://raw.githubusercontent.com/tableflowhq/db-webhooks/main/{.env,docker-compose.yml,.dockerignore,frontend.env}
docker-compose up -d
```

## Features

### Template Strings

Embed variables from database rows in your actions
When adding an action, you can insert data from the row into the response body of the POST request by using template
strings.

For instance, if your table has a column called `email`, you would put the value `${email}` in the request
body: `{"text":"User created: ${email}!"}`

The prefixes `new.` and `old.` can be used if a new (INSERT, UPDATE) or old (UPDATE, DELETE) row is available. If a
prefix is not specified, the new or old values will be used depending on the event.
Example: `{"text":"User updated: ${old.email} is now ${new.email}!"}`

Meta values can also be used to get more query information. The following are available:

1. `meta.table` (table name)
2. `meta.schema` (schema name)
3. `meta.event` (INSERT, UPDATE, or DELETE)
4. `meta.user` (the Postgres user who ran the query that triggered the trigger)
5. `meta.event_summary` (a formatted summary of the user, table, and event)
6. `meta.changed` (the values changed for an UPDATE)

## Postgres Configuration

### Create a Postgres User

When setting up your database in DB Webhooks, you'll need a Postgres user to connect to the database. You can either use
an existing user or create a new one. This guide shows how to create a new Postgres user with the correct permissions
needed for DB Webhooks.

If you're using an existing user, make sure the user has the `create` permission on the schema(s) you want to use.

**Note:** If you want DB Webhooks to also clean up the triggers and functions it creates when all actions are removed on
a table, the user needs to be an owner as Postgres does not have a "drop" permission.

### Why is the `create` Permission Needed?

DB Webhooks uses the create permission to create triggers and functions. A trigger (and corresponding function) is
created whenever a new action on a table is added which doesn't have the DB Webhooks trigger already.

DB Webhooks will delete the trigger and function it created if the last action on a table is removed, so it doesn't
leave any extra triggers on your database.

You can see how the trigger works by looking at its implementation in `util.go`.

#### 1. Create the User

```sql
create user db_webhooks with encrypted password 'XXXXXXXXXXXXXXX';
```

#### 2. Grant Create Access

```sql
grant create on schema public to db_webhooks;
-- If you have tables in a schema other than `public`, add the schema here
grant create on schema your_other_schema_if_needed to db_webhooks;
```
