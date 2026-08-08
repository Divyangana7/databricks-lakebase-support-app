# Internal Support App — setup & deployment

A Flask + Lakebase app for the Day 1 homework. It reuses the course boilerplate's
`lakebase.py` connection helper unchanged and replaces the application logic with
the support-ticket scenario.

## Files

| File | Purpose |
| --- | --- |
| `app.py` | Flask app: list tickets, view a ticket, create tickets, add messages, update status |
| `lakebase.py` | Lakebase connection helper (reused verbatim from the course repo) |
| `schema.sql` | Table DDL + required sample data (3 tickets, 2+ messages each, 3 statuses) |
| `templates/` | `base.html`, `index.html`, `ticket.html` — the UI |
| `app.yaml` | Databricks Apps run config |
| `requirements.txt` | Python dependencies |

## Data model

- **tickets**: `ticket_id`, `title`, `status`, `created_by`, `created_at`
- **ticket_messages**: `message_id`, `ticket_id`, `message_text`, `author`, `created_at`
- `ticket_messages.ticket_id` is a foreign key referencing `tickets.ticket_id`
  (`ON DELETE CASCADE`), which satisfies the "must reference a ticket" requirement.
- `status` is constrained to `open`, `in_progress`, or `resolved`.

## One-time setup

### 1. Create the Lakebase instance and a password role
Follow the course steps: create a Lakebase instance, enable native (password)
authentication, create a role, and copy the connection URL, which looks like:

```
postgresql://<role>:<password>@<host>.database.cloud.databricks.com:5432/databricks_postgres?sslmode=require
```

### 2. Store the connection URL as a secret
`lakebase.py` reads the URL from the Databricks secret scope `database`, key
`lakebase-url` (base64-decoded). Use the course's `setup_secrets.py`, or create it
directly, e.g. from a notebook:

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
w.secrets.create_scope("database")   # skip if it already exists
w.secrets.put_secret(
    scope="database",
    key="lakebase-url",
    string_value="postgresql://<role>:<password>@<host>...:5432/databricks_postgres?sslmode=require",
)
```

> The app that runs in Databricks Apps must have permission to read this scope.
> The service principal created for your app needs `READ` on the `database` scope.

### 3. Create the schema and sample data
Open the Databricks SQL editor connected to your Lakebase instance (or a `psql`
session using the URL) and run **`schema.sql`**. It creates both tables and inserts
the sample data only if the tables are empty, so it is safe to run more than once.

(The app also runs the `CREATE TABLE IF NOT EXISTS` statements on startup, so the
tables exist even before you run `schema.sql`; running `schema.sql` is what loads
the required sample rows.)

## Run locally (optional)

```bash
pip install -r requirements.txt
python app.py
```

Local runs authenticate to Databricks (to fetch the secret) using your Databricks
CLI / SDK credentials. Then open http://localhost:8000.

## Deploy on Databricks Apps (UI, no CLI)

1. **Workspace → Create → Git folder**, point it at your fork of this repo, and
   create the folder. This is the app's source.
2. **Compute → Apps → Create app → Custom.** Name it (e.g. `internal-support`).
3. When asked for source, select the Git folder containing `app.py` and `app.yaml`.
4. Grant the app's service principal `READ` on the `database` secret scope
   (Databricks may prompt you to add the secret as an app resource).
5. Click **Deploy**. Databricks reads `app.yaml`, runs `python app.py`, and serves
   on the injected port.
6. Open the app URL. Confirm:
   - existing tickets load from Lakebase,
   - a new ticket can be created,
   - a message can be added,
   - a ticket's status can be updated,
   - changes persist after a refresh.

To redeploy after code changes: pull the latest into the Git folder, then click
**Deploy** again.

## Notes / places you may want to extend

- Right now anyone can type any name in the "Your name" / author fields. In a real
  app you would use the signed-in Databricks user (available via app headers).
- There is no pagination on the ticket list; fine for the homework, worth adding
  for larger datasets.
- If you later enable Change Data Feed (CDF), run
  `ALTER TABLE tickets REPLICA IDENTITY FULL;` and the same for `ticket_messages`.
