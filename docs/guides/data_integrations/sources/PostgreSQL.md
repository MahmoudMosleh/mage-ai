# Change Data Capture (CDC) with PostgreSQL

Mage supports 2 types of change data capture with PostgreSQL:

1. Batch query
1. Log replication

![](https://user-images.githubusercontent.com/78053898/198754309-2ef713a7-62c8-4ea8-9ebb-8c24ed038cb3.png)

## Batch query

Mage will query PostgreSQL in batches using `SELECT`, `WHERE`, and `ORDER BY`
statements.

## Log replication

Mage will read the logs from PostgreSQL and use those as instructions to either
create new rows, update existing rows, or delete rows in the destination.

### How to setup log replication with PostgreSQL

#### Setup in PostgreSQL

1. Open the `postgresql.conf` file. Here is an example location on Mac OSX:
`/Users/Mage/Library/Application Support/Postgres/var-14/postgresql.conf`.
1. Under the settings section, change the value of `wal_level` to `logical`. The line in your
`postgresql.conf` file should look like this:
    ```text
    wal_level = logical
    ```
1. Restart the PostgreSQL service or database. You can do this via the PostgreSQL app or if you’re
on Linux, run the following commands:
    ```bash
    sudo service postgresql stop
    sudo service postgresql start
    ```
1. Run the following query in your PostgreSQL database: `SHOW wal_level`. The result should be:

    | `wal_level` |
    | --- |
    | `logical` |

1. Run the following command in PostgreSQL to create a replication slot:
    ```sql
    SELECT pg_create_logical_replication_slot('mage_slot', 'pgoutput');
    ```
    
    <sub>`mage_slot` is used in Mage’s [code](https://github.com/mage-ai/mage-ai/blob/master/mage_integrations/mage_integrations/sources/postgresql/__init__.py#L85)</sub>

    The result should looking something like this:

    | `pg_create_logical_replication_slot` |
    | --- |
    | `(mage_slot,0/51A80778)` |

1. Create a publication for all tables or for 1 specific table using the following commands:
    ```sql
    CREATE PUBLICATION mage_pub FOR ALL TABLES;
    ```

    <sub>`mage_pub` is used in Mage’s [code](https://github.com/mage-ai/mage-ai/blob/master/mage_integrations/mage_integrations/sources/postgresql/__init__.py#L126)</sub>


    or for 1 table:

    ```sql
    CREATE PUBLICATION mage_pub FOR TABLE some_schema.some_table_name;
    ```

    <sub>Replace `some_schema` with the schema of the table and `some_table_name` with the name
    of the table you want to replicate.</sub>

1. Verify that the publication was created successfully by running the following command in PostgreSQL:
    ```sql
    SELECT * FROM pg_publication_tables;
    ```

    The result should looking something like this:

    | `pubname` | `schemaname` | `tablename` |
    | --- | --- | --- |
    | `mage_pub` | `public` | `users` |

<br />

#### Create data integration pipeline in Mage

Follow this [guide to create a data integration pipeline](../../pipelines/DataIntegrationPipeline.md)
in Mage.

However, choose <b>PostgreSQL</b> as the source and choose <b>`LOG_BASED`</b> as the
replication method.

<br />

#### Testing pipeline end-to-end

Once you’ve created the pipeline, add a few rows into your PostgreSQL table that you just
created a logical replication for.

You can use the `INSERT` command to add rows. For example:

```sql
INSERT INTO some_schema.some_table_name
VALUES (1, 2, 3)
```

<sub>
  Replace `some_schema` with the schema of the table and `some_table_name` with the name
  of the table you want to replicate.
</sub>

<sub>
  Change the `VALUES` to match the columns in your table.
</sub>

##### Verify replication logs being created

Run the following commands in PostgreSQL to check for new logs:

```sql
SELECT
  *
FROM pg_logical_slot_peek_binary_changes('mage_slot', null, null, 'proto_version', '1', 'publication_names', 'mage_pub');
```

##### Run sync

After you added a few new rows, [create a trigger](../../../tutorials/triggers/schedule.md)
to start running your pipeline and begin syncing data.

<br />
