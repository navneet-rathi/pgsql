# Load CSV Data into a Remote PostgreSQL (Containerized) with Ansible

This playbook loads a local CSV file (`userdata.csv`) into a PostgreSQL database that runs **inside a container on a remote host**, where we have **no shell or filesystem access to the container**. The only access we have is the network connection to PostgreSQL (`192.168.64.41:5432`).

## What it does

1. Creates the database `patching` (if missing).
2. Creates the table `userdata` (if missing).
3. Reads `userdata.csv` on the machine running Ansible (the controller / AAP execution environment).
4. Inserts the rows into `userdata` in batches over the normal PostgreSQL connection.

## Why not `postgresql_copy`?

`community.postgresql.postgresql_copy` with `copy_from:` runs a **server-side** `COPY ... FROM '/path/file'`. The PostgreSQL server process itself opens the file, so:

- The file must exist **inside the container's filesystem**. Our `userdata.csv` only exists on the Ansible host, so the load fails.
- The DB user also needs superuser rights or the `pg_read_server_files` role.

Since we cannot copy files into the container or mount a volume, server-side `COPY` is not an option.

| Approach | Where the file is read | Needs file in container? | Result |
|---|---|---|---|
| `postgresql_copy` (`copy_from`) | PostgreSQL server (container) | Yes | Not working |
| `psql \copy` | Client machine (Ansible host) | No | Working |
| `read_csv` + `postgresql_query` (this playbook) | Client machine (Ansible host) | No | Working, no `psql` needed |

`psql \copy` works because it is **client-side**: `psql` reads the local file and streams the rows over the connection. The playbook below does the same thing with pure Ansible modules, so it does not depend on the `psql` binary being installed.

## Requirements

On the machine (or AAP execution environment) that runs the playbook:

- Ansible collections: `community.postgresql`, `community.general`
- Python package: `psycopg2` (or `psycopg2-binary`)
- Network access to `192.168.64.41:5432`
- A PostgreSQL user allowed to create databases/tables and insert data, and a `pg_hba.conf` rule that allows this host to connect

```bash
ansible-galaxy collection install community.postgresql community.general
pip install psycopg2-binary
```

> In AAP, these collections and `psycopg2` must be present in the **execution environment** used by the job template.

## CSV format

The file must have a header row. The playbook expects these columns:

```csv
Id,fname,lname
1,John,Doe
2,Jane,Smith
```

The path in `local_csv_path` is resolved relative to the playbook / project directory. In AAP, commit the CSV to the project (or use an absolute path available in the execution environment).

## Playbook

Credentials are **not** hard-coded. Supply them through Ansible Vault or an AAP credential / extra vars (see [Credentials](#credentials)).

```yaml
---
- name: Load CSV data into remote PostgreSQL running in a container
  hosts: localhost
  become: false
  gather_facts: false
  collections:
    - community.postgresql

  vars:
    # pg_user and pg_password come from Vault / AAP credential / extra vars
    db_host: 192.168.64.41
    db_port: 5432
    db_name: patching
    table_name: userdata
    local_csv_path: userdata.csv
    chunk_size: 500

  tasks:
    - name: Create the database
      community.postgresql.postgresql_db:
        name: "{{ db_name }}"
        state: present
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        login_user: "{{ pg_user }}"
        login_password: "{{ pg_password }}"
      no_log: true

    - name: Create the table
      community.postgresql.postgresql_table:
        name: "{{ table_name }}"
        state: present
        columns:
          - id serial primary key
          - fname text
          - lname text
        login_db: "{{ db_name }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        login_user: "{{ pg_user }}"
        login_password: "{{ pg_password }}"
      no_log: true

    - name: Read local CSV file into memory
      community.general.read_csv:
        path: "{{ local_csv_path }}"
      register: csv_data

    - name: Bulk insert rows in batches (one query per chunk)
      community.postgresql.postgresql_query:
        login_db: "{{ db_name }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        login_user: "{{ pg_user }}"
        login_password: "{{ pg_password }}"
        query: |
          INSERT INTO {{ table_name }} (id, fname, lname)
          VALUES
          {% for row in item %}
          ({{ row.Id | int }}, '{{ row.fname | replace("'", "''") }}', '{{ row.lname | replace("'", "''") }}'){{ "," if not loop.last else "" }}
          {% endfor %};
      loop: "{{ csv_data.list | batch(chunk_size) | list }}"
      loop_control:
        label: "Inserting a chunk of up to {{ chunk_size }} rows"
      no_log: true
```

### How the batching works

`csv_data.list | batch(chunk_size)` splits the rows into groups of 500. The task loops over the groups and sends **one multi-row `INSERT` per group**, which is far faster than one insert per row and keeps each query a reasonable size.

## Running it

```bash
ansible-playbook load_userdata.yml -e @secrets.yml --ask-vault-pass
```

Where `secrets.yml` is encrypted with `ansible-vault` and contains:

```yaml
pg_user: nrathi
pg_password: <your password>
```

In AAP: create a **Custom Credential Type** (or use extra vars with the survey password field) for `pg_user` / `pg_password`, attach it to the job template, and make sure the execution environment has the requirements above.

## Credentials

Do **not** commit database passwords to Git or leave them in plain text in playbooks. Use one of:

- `ansible-vault encrypt_string` for individual values
- An encrypted vars file (`ansible-vault encrypt secrets.yml`)
- An AAP credential injected as `pg_user` / `pg_password`

If a password has already been committed or shared, **rotate it**.

## Verifying the load

```bash
psql -h 192.168.64.41 -U <user> -d patching -c "SELECT count(*) FROM userdata;"
```

or in Ansible:

```yaml
- name: Count rows
  community.postgresql.postgresql_query:
    login_db: "{{ db_name }}"
    login_host: "{{ db_host }}"
    login_user: "{{ pg_user }}"
    login_password: "{{ pg_password }}"
    query: "SELECT count(*) AS total FROM {{ table_name }};"
  register: result
  no_log: true
```

## Notes and gotchas

- **Re-running inserts duplicates / fails.** Because `id` is a primary key, running the load twice fails on duplicate ids. Truncate first (`TRUNCATE userdata;`) or add `ON CONFLICT (id) DO NOTHING` to the query.
- **Serial sequence.** The playbook inserts explicit `id` values, so the `serial` sequence is not advanced. If you later insert without an `id`, run `SELECT setval(pg_get_serial_sequence('userdata','id'), (SELECT max(id) FROM userdata));` first. Alternatively, drop `id` from the `INSERT` and let the database generate it.
- **Header case matters.** `row.Id`, `row.fname`, `row.lname` must match the CSV header exactly (case-sensitive).
- **Quoting.** Single quotes in names (for example `O'Brien`) are escaped by the `replace("'", "''")` filter. For untrusted data, prefer `positional_args` with parameterized queries, at the cost of more round trips.
- **Very large files.** `read_csv` loads the whole file into memory. For files with millions of rows, use `psql \copy` through the `command` module instead.
- **Connectivity.** If the connection is refused, check that the container publishes (or uses host networking for) port 5432, that `listen_addresses` includes the host's interface, and that `pg_hba.conf` allows the client IP.

## Alternative: `psql \copy` (also works)

```yaml
- name: Load data with client-side \copy
  ansible.builtin.command: >
    psql -h {{ db_host }} -U {{ pg_user }} -d {{ db_name }}
    -c "\copy {{ table_name }} FROM '{{ local_csv_path }}' WITH (FORMAT csv, HEADER true, DELIMITER ',')"
  environment:
    PGPASSWORD: "{{ pg_password }}"
  no_log: true
```

Requires the `psql` client on the machine running the playbook. Note that the CSV in this case must have columns in the order of the table, or you must list the columns explicitly.
