# Backup and Recovery for Appointal

A **backup** is a saved copy of the database. **Recovery (restore)** means putting that copy back after data is lost or damaged. Replication protects against server failure. Backups protect against mistakes (accidental deletes, bad updates).

## Tools
`mongodump` and `mongorestore` are part of the **MongoDB Database Tools**. They are a separate download from the MongoDB website. After installing on Windows, check with:
```
mongodump --version
```
If the command isn't found, add the Database Tools `bin` folder to your Windows PATH.

## Backup with mongodump
```
mongodump --uri="mongodb+srv://<username>:<password>@<cluster-host>/" --db=appointal --out=backup\2026-10-29
```
| Part | Meaning |
|---|---|
| `--uri` | The connection string (placeholders here, use your real one locally) |
| `--db=appointal` | Back up only the `appointal` database |
| `--out=...` | Folder where the backup is saved |

Use a connection string **without** a database name in the path, so it doesn't conflict with `--db`.

The backup folder will contain `appointal\doctors.bson`, `patients.bson`, `appointments.bson`, plus `.metadata.json` files (which also store the index definitions).

## Restore with mongorestore
```
mongorestore --uri="mongodb+srv://<username>:<password>@<cluster-host>/" --nsInclude="appointal.*" --drop backup\2026-10-29
```
| Part | Meaning |
|---|---|
| `--nsInclude="appointal.*"` | Restore only the `appointal` database's collections |
| `--drop` | Delete the existing collection first, then restore it (avoids duplicates). Remove it if you want to add to existing data |
| last argument | The backup folder created by mongodump |

## Practice exercise (good for the viva)
1. Add some doctors, patients and appointments in the app.
2. Run `mongodump` as shown above.
3. Delete a few records (or a whole collection) in the app or Atlas.
4. Run `mongorestore` with `--drop`.
5. Refresh the app. The data is back.

## Suggested backup strategy
- Take a backup **before** big changes and **regularly** (for example daily).
- Name folders with the date.
- Store copies **outside** the project folder, on another drive or cloud storage. Don't commit them to Git (`backup/` is in `.gitignore`).
- Backup files contain real data, so protect them.
- **Test your restore** occasionally. A backup you've never restored is not proven.

## Atlas built-in backups
Paid Atlas clusters offer automated Cloud Backups with point-in-time restore. Free-tier clusters generally don't include them, so use `mongodump` for this project. Check Atlas's current documentation for exact tier features.

## Security reminder
Never write real usernames/passwords into scripts or documents that you share or upload to GitHub.
