# Project notes for Claude

## Database: Nexus on Omni (changed 2026-11-06)

- Titan is moving all Anvil apps to the in-house Anvil App Server, **Omni** (192.168.16.173), with a local database.
- All app changes from now on must reference the **`nexus` database on Omni (192.168.16.173)**. Do not target the old Nexus server, server6 (144.202.5.215), in new code, SQL scripts, docs, or secret examples.
- This app has no server code yet. When database access is added, follow the standard `db_config.py` pattern used by the other Titan apps (Anvil Secrets `NEXUS_SERVER` etc.), pointing at Nexus on Omni.
