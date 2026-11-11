# Campus-Central-

"""FreeFood@UMass - find free food and free stuff from campus clubs.

Run:  pip install flask  &&  python app.py   ->  http://localhost:5000
"""
import hashlib
import json
import os
import sqlite3
from datetime import datetime

from flask import Flask, jsonify, redirect, render_template_string, request

from classifier import classify
from sources import load_all

DB = os.environ.get("FREEFOOD_DB", "freefood.db")
app = Flask(__name__)

SCHEMA = """CREATE TABLE IF NOT EXISTS posts(
  id TEXT PRIMARY KEY, org TEXT, title TEXT, text TEXT, url TEXT, source TEXT,
  start TEXT, location TEXT, score REAL, items TEXT, created TEXT)"""


def db():
    c = sqlite3.connect(DB)
    c.row_factory = sqlite3.Row
    c.execute(SCHEMA)
    return c


def save(post: dict) -> bool:
    """Classify and store a post. Returns True if it contains free food/stuff."""
    r = classify(post["text"])
    if not r["has_free_food"]:
        return False
    pid = hashlib.sha1((post["org"] + post["text"]).encode()).hexdigest()[:16]
    items = ", ".join(r["food_items"] + r["free_stuff"]) or r.get("llm_what", "")
    with db() as c:
        c.execute(
            "INSERT OR REPLACE INTO posts VALUES(?,?,?,?,?,?,?,?,?,?,?)",
            (pid, post["org"], post["title"], post["text"], post.get("url", ""),
             post.get("source", "submit"), post.get("start"), post.get("location", ""),
             r["score"], items, datetime.now().isoformat()),
        )
    return True


def upcoming(limit=100):
    now = datetime.now().isoformat()
    with db() as c:
        rows = c.execute(
            "SELECT * FROM posts WHERE start IS NULL OR start >= ? "
            "ORDER BY start IS NULL, start LIMIT ?", (now, limit)).fetchall()
    return [dict(r) for r in rows]


PAGE = """<!doctype html><meta name=viewport content="width=device-width,initial-scale=1">
<title>FreeFood@UMass</title>
<style>
body{font:16px system-ui;max-width:760px;margin:0 auto;padding:16px;background:#fafafa;color:#222}
h1{color:#881c1c;margin-bottom:0}.sub{color:#666;margin-top:4px}
.card{background:#fff;border:1px solid #e5e5e5;border-radius:12px;padding:14px;margin:12px 0}
.org{font-weight:700}.tag{display:inline-block;background:#fde8e8;color:#881c1c;border-radius:99px;
padding:2px 10px;font-size:13px;margin:6px 6px 0 0}.meta{color:#666;font-size:14px}
form{background:#fff;border:1px solid #e5e5e5;border-radius:12px;padding:14px;margin-top:24px}
input,textarea{width:100%;box-sizing:border-box;margin:4px 0 10px;padding:8px;font:inherit}
button{background:#881c1c;color:#fff;border:0;border-radius:8px;padding:10px 16px;font:inherit}
</style>
<h1>🍕 FreeFood@UMass</h1>
<p class=sub>Free food & free stuff from campus clubs, found for you.</p>
{% for p in posts %}
<div class=card>
  <div class=org>{{p.org}}</div>
  <div>{{p.title}}</div>
  <div class=meta>{% if p.start %}🕒 {{p.start.replace('T',' ')[:16]}}{% else %}🕒 time: see post{% endif %}
  {% if p.location %} · 📍 {{p.location}}{% endif %}</div>
  {% for i in p['items'].split(', ') if i %}<span class=tag>{{i}}</span>{% endfor %}
  {% if p.url %}<div class=meta><a href="{{p.url}}">original post</a></div>{% endif %}
</div>
{% else %}<p>No free food found yet. Check back soon!</p>{% endfor %}
<form method=post action=/submit>
  <b>Spotted free food? Add it.</b>
  <input name=org placeholder="Club / organization" required>
  <input name=location placeholder="Location (e.g. Campus Center 163)">
  <input name=start type=datetime-local>
  <textarea name=text rows=3 placeholder="Paste the post or describe the event" required></textarea>
  <button>Submit</button>
</form>"""


@app.get("/")
def home():
    return render_template_string(PAGE, posts=upcoming())


@app.get("/api/events")
def api_events():
    return jsonify(upcoming())


@app.post("/submit")
def submit():
    f = request.form
    save({"org": f["org"], "title": f["text"][:80], "text": f["text"],
          "location": f.get("location", ""), "start": f.get("start") or None,
          "source": "student-submitted"})
    return redirect("/")


@app.post("/refresh")
def refresh():
    cfg = json.load(open("sources.json")) if os.path.exists("sources.json") else {}
    posts, errors = load_all(cfg)
    found = sum(save(p) for p in posts)
    return jsonify({"scanned": len(posts), "free_food_found": found, "errors": errors})


if __name__ == "__main__":
    app.run(debug=True)
