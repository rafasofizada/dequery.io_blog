---
title: "Postgres: disable, reenable and revalidate foreign key constraints"
summary: >
  - Why would someone need to do this?
  
  - How to toggle foreign key constraints, without `TRIGGER ALL`?
  
  - How to validate constraints after reenabling them?
date: 2026-04-16T15:49:05+02:00
slug: 2026-04-16-pg_constraint_validation
type: posts
draft: false
categories:
  - postgres
tags:
  - postgres
  - sql
---

Say you’re writing an ETL tool for Postgres [article coming soon]. If you’re not loading all the data in one transaction, and in an arbitrary order, even with `DEFERRED`, you’ll get foreign key constraint violations.

So you need to temporarily disable all foreign key constraints, load the data, enable them back, and, optionally, revalidate all constraints.

### How to disable foreign key constraints in PostgreSQL?

- Foreign key constraints are implemented using **triggers**
- Hence, to temporarily disable foreign key constraints, you'd need to disable the triggers:
    
    ```sql
    -- Disables all triggers
    ALTER TABLE your_table_name DISABLE TRIGGER ALL;
    
    -- Re-enable constraints when finished
    ALTER TABLE your_table_name ENABLE TRIGGER ALL;
    ```
    
- Alternative, `session_replication_role`
    
    ```sql
    -- Disables all triggers
    SET session_replication_role = replica;
    
    -- Re-enable constraints when finished
    SET session_replication_role = origin;
    ```
    
    - [What does `session_replication_role` have to do with triggers?](https://postgrespro.com/list/thread-id/2369438)
- However, this disables *all* triggers, including user-defined ones (e.g. `BEFORE INSERT`, `AFTER UPDATE`, etc.)

> But I have user-defined triggers that I want to run while data is loaded!
> 

### **How to disable** *only* **foreign-key-related triggers?**

- There’s no single DDL command that allows you this.
- However, you can query the system trigger names that implement the foreign key constraints. Then, run a `PL/pgSQL` procedure to disable/enable them.

```sql
DO $$
DECLARE
  trg RECORD;
BEGIN
  FOR trg IN
    SELECT
      quote_ident(nspname)      AS schema_name,
      quote_ident(rel.relname)  AS table_name,
      quote_ident(t.tgname)     AS trigger_name,
      /* OPTIONAL */
      -- Whether trigger is enabled in human-readable form
      ---- Why 'O' and not 'E'? tgenabled is not a simple on/off, it's related to replication role settings: O – origin, R - replica, A – always, D – disabled
      CASE t.tgenabled
        WHEN 'O' THEN 'ENABLED'
        WHEN 'D' THEN 'DISABLED'
        ELSE 'UNKNOWN'
      END AS trigger_status,
      -- Events the trigger is set on
      ARRAY(
        SELECT "event"
        -- tgtype is a bitmask.
        -- Meaning of each bit: https://github.com/postgres/postgres/blob/master/src/include/catalog/pg_trigger.h#L95
        FROM unnest(ARRAY['INSERT', 'DELETE', 'UPDATE', 'TRUNCATE']) WITH ORDINALITY AS u("event", idx)
        WHERE ((t.tgtype >> (u.idx::INT + 1)) & 1) != 0
      ) AS trigger_events,
      -- Function the trigger implements (e.g. "RI_FKey_check_ins", "RI_FKey_check_upd")
      pg_proc.proname AS function_name,
      -- Human-readable description of the constraint
      pg_get_constraintdef(con.oid) AS constraint_description
      /* OPTIONAL */
  FROM
    pg_trigger t
    -- table
    JOIN pg_class rel
      ON t.tgrelid = rel.oid
    -- schema
    JOIN pg_namespace
      ON pg_class.relnamespace = pg_namespace.oid
    -- constraints (that the trigger is implementing (optional, as described in WHERE))
    JOIN pg_constraint con
      ON con.conrelid = rel.oid
      AND t.tgconstraint = con.oid
    -- function (that the trigger is implementing (optional, as described in SELECT))
    JOIN pg_proc
      ON pg_proc.oid = t.tgfoid
    WHERE
      t.tgisinternal -- is system-defined (e.g. excluding set__key in triggers.sql)
      AND t.tgenabled = -- 'O' for enabled, 'D' for disabled
      AND con.contype = 'f' -- is foreign key constraint...
      -- ...(redundant, because only fkey constrs are implemented with triggers, but to be safe)
  LOOP
    -- If enabling...
    EXECUTE format('ALTER TABLE %I.%I ENABLE TRIGGER %I', trg.schema_name, trg.table_name, trg.trigger_name);
    RAISE NOTICE 'Enabled system trigger ''%'' for table ''%'' on ''%'' events for constr ''%''', trg.trigger_name, trg.table_name, trg.trigger_events, trg.constraint_description;
    -- If disabling...
    EXECUTE format('ALTER TABLE %I.%I DISABLE TRIGGER %I', trg.schema_name, trg.table_name, trg.trigger_name);
    RAISE NOTICE 'Disabled system trigger ''%'' for table ''%'' on ''%'' events for constr ''%''', trg.trigger_name, trg.table_name, trg.trigger_events, trg.constraint_description;
  END LOOP;
END $$
```

<aside>
⚠️

**Foreign key constraints aren’t validated when you enable them back on.** At this point, you might have data that violates constraints, but you’ll get no errors.

</aside>

> Well that’s…bad. I still want to make sure the data I’ve loaded satisfies all of my constraints.
> 

### How to validate foreign key constraints after re-enabling them?

- Easy!

```sql
ALTER TABLE your_table_name VALIDATE CONSTRAINT fk_constraint_name;
```

- … nope ❌ . [`VALIDATE CONSTRAINT` only works on constraints marked as `NOT VALID`](https://www.postgresql.org/docs/current/sql-altertable.html#SQL-ALTERTABLE-DESC-VALIDATE-CONSTRAINT).
- You can’t `ALTER CONSTRAINT`  to make it `NOT VALID` – `NOT VALID` [is only available at constraint-creation-time.](https://www.notion.so/How-to-disable-reenable-and-revalidate-foreign-key-constraints-in-Postgres-1d3f1b4d927380c288c4c4f06d52898d?pvs=21)
- Now, you have 2 options:
    - Drop and recreate all constraints in a `PL/pgSQL` procedure (difficult).
    - Or…

### `VALIDATE CONSTRAINT` system catalog hack

We can fool the `VALIDATE CONSTRAINT` command by manually setting a flag (responsible for `NOT VALID`) in the system catalogs:

<aside>
⚠️

Common advice is to refrain from ever manually editing system catalogs. However, in 1 year of using this in our in-house Postgres ETL tool, we’ve had 0 problems with this approach.

</aside>

```sql
DO $$
DECLARE
  r RECORD;
BEGIN
  FOR r IN (
    SELECT
      conrelid::regclass AS table_name,
      conname AS constraint_name
    FROM pg_constraint
    WHERE contype = 'f'  -- foreign key constraints
  )
  LOOP
    UPDATE pg_constraint
    SET convalidated = false -- the 'NOT VALID' flag
    WHERE
      conrelid = r.table_name::regclass
      AND conname = r.constraint_name;

    RAISE NOTICE 'Set constraint % on table % to NOT VALID', r.constraint_name, r.table_name;
  END LOOP;
END $$;
```

Now, if you run:

```sql
DO $$
DECLARE
  fk RECORD;
BEGIN
  FOR fk IN
    SELECT conname, conrelid::regclass AS table_name
    FROM pg_constraint
    WHERE contype = 'f' AND convalidated = FALSE
  LOOP
    EXECUTE format('ALTER TABLE %s VALIDATE CONSTRAINT %I', fk.table_name, fk.conname);
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

Your constraints are validated, and you’ll get errors if any data fails them.
