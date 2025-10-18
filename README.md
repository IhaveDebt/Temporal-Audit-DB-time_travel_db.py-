#!/usr/bin/env python3
"""
Temporal Audit DB - simple time-travel layer on top of SQLite (prototype).
- Stores row versions on write and offers time-travel queries.
Run: python3 src/time_travel_db.py
"""
import sqlite3, time, json, os
from typing import Any, Dict, List, Tuple

DB = "time_travel.db"

def init_db():
    if os.path.exists(DB):
        os.remove(DB)
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("CREATE TABLE items(id TEXT PRIMARY KEY, data TEXT)")
    c.execute("CREATE TABLE history(row_id TEXT, ts INTEGER, data TEXT)")
    conn.commit()
    return conn

class TimeTravelDB:
    def __init__(self, conn):
        self.conn = conn

    def upsert(self, row_id: str, data: Dict[str, Any]):
        ts = int(time.time())
        cur = self.conn.cursor()
        cur.execute("INSERT OR REPLACE INTO items(id,data) VALUES(?,?)", (row_id, json.dumps(data)))
        cur.execute("INSERT INTO history(row_id, ts, data) VALUES(?,?,?)", (row_id, ts, json.dumps(data)))
        self.conn.commit()
        print(f"Upserted {row_id} @ {ts}")

    def get(self, row_id: str) -> Dict[str, Any]:
        cur = self.conn.cursor()
        cur.execute("SELECT data FROM items WHERE id=?", (row_id,))
        r = cur.fetchone()
        return json.loads(r[0]) if r else None

    def travel(self, row_id: str, ts: int) -> Dict[str, Any]:
        cur = self.conn.cursor()
        cur.execute("SELECT data FROM history WHERE row_id=? AND ts<=? ORDER BY ts DESC LIMIT 1", (row_id, ts))
        r = cur.fetchone()
        return json.loads(r[0]) if r else None

    def diffs(self, row_id: str) -> List[Tuple[int, Dict[str,Any]]]:
        cur = self.conn.cursor()
        cur.execute("SELECT ts, data FROM history WHERE row_id=? ORDER BY ts ASC", (row_id,))
        return [(r[0], json.loads(r[1])) for r in cur.fetchall()]

def demo():
    conn = init_db()
    db = TimeTravelDB(conn)
    db.upsert("u1", {"name":"Alice","role":"engineer"})
    time.sleep(1)
    db.upsert("u1", {"name":"Alice","role":"senior engineer"})
    time.sleep(1)
    db.upsert("u1", {"name":"Alice","role":"team lead"})
    print("Current:", db.get("u1"))
    history = db.diffs("u1")
    print("History:", history)
    mid_ts = history[1][0]
    print("Travel to ts", mid_ts, "=>", db.travel("u1", mid_ts))

if __name__ == "__main__":
    demo()
