# Exam Manager — web edition (Flask)

A browser version of the examination module: students, exams, subjects, exam
schedule, marks entry and results — deployable on Render (or any Python host)
so it can be opened from any device with a link.

## Files

```
app.py               the whole application (routes + database + logic)
templates/           HTML pages (Jinja)
static/style.css     styling
static/icon.png      favicon
requirements.txt     Flask + gunicorn
Procfile             start command for Render/Heroku-style hosts
render.yaml          one-click Render blueprint
```

## Why your first deploy failed

The earlier `app.py` could not run on Render for three separate reasons:

1. **No `requirements.txt`** — Render installs dependencies from it. Without
   the file Flask is not installed, so the app dies on `import flask`.
2. **No start command** — Render has to know how to start the service
   (`gunicorn app:app`). A bare `python app.py` with the development server is
   not enough.
3. **`app.run(debug=True)`** — the Flask dev server binds to `127.0.0.1:5000`,
   which is only reachable inside the container. Render needs the app to listen
   on `0.0.0.0` and on the port given in `$PORT`, otherwise it reports
   “no open ports detected”.

All three are fixed here.

## Deploy on Render — step by step

1. **Push this folder to GitHub** (repo root = the folder containing `app.py`).

   ```bash
   git init
   git add .
   git commit -m "Exam Manager web edition"
   git branch -M main
   git remote add origin https://github.com/<you>/<repo>.git
   git push -u origin main
   ```

2. **Render → New → Web Service** → connect that repository.

3. Fill in:

   | Field | Value |
   |---|---|
   | Runtime | Python 3 |
   | Build command | `pip install -r requirements.txt` |
   | Start command | `gunicorn app:app --bind 0.0.0.0:$PORT --workers 2 --timeout 120` |
   | Instance type | Free |

4. **Environment variables** (Advanced → Add Environment Variable):

   | Key | Value |
   |---|---|
   | `APP_PASSWORD` | a password only you and the staff know — **set this** |
   | `SECRET_KEY` | click *Generate* (any long random string) |
   | `SCHOOL_NAME` | e.g. `Jindal Public School` |

5. **Create Web Service** → wait for the build → open the URL.

`render.yaml` is included, so you can also use **New → Blueprint** and Render
reads the build/start commands and variables by itself.

## Important: where the data lives

The app stores everything in a SQLite file. On Render's **free** plan the
filesystem is temporary — **the database is wiped whenever the service is
redeployed or restarted**. Three options:

- **Keep it simple for a demo** — fine, just export the CSVs before you redeploy.
- **Add a Render Disk** (paid, a few dollars a month): create a disk mounted at
  `/var/data`, then set the environment variable `DB_PATH=/var/data/exam_manager.db`.
  Data then survives restarts.
- **Move to PostgreSQL** later — the queries are plain SQL and the schema is in
  `app.py`; that is a small change if you need it.

Also on the free plan the service sleeps after ~15 minutes without traffic, so
the first visit after a break takes a few seconds to wake up.

## Run it on your own computer first

```bash
pip install -r requirements.txt
python app.py                 # http://127.0.0.1:5000
```

Optional:

```bash
APP_PASSWORD=mypass SCHOOL_NAME="Jindal Public School" python app.py
```

## Security checklist before sharing the link

- [ ] `APP_PASSWORD` is set (otherwise anyone with the URL sees student data)
- [ ] `SECRET_KEY` is set and random
- [ ] The link is shared only with school staff, not posted publicly
- [ ] You are aware the free plan can reset the database — export CSVs regularly
- [ ] Student data on a hosted server is your school's responsibility under
      India's DPDP Act, 2023; the app keeps it on the server you chose and does
      not send it anywhere else

## Troubleshooting: “Internal Server Error” (HTTP 500)

A 500 means the service started but a page raised an error. Open this URL on
your live app:

```
https://<your-app>.onrender.com/healthz
```

It prints a JSON checklist — templates present, CSS present, database
writable, password set — so you can see instantly what is missing. The error
page itself now also lists the same checks plus the traceback instead of the
blank “Internal Server Error”.

Most common causes:

| Symptom in `/healthz` | Cause | Fix |
|---|---|---|
| `templates folder … missing: base.html, …` | the `templates/` folder was not uploaded | the app moves loose html files into `templates/` on boot, so a fresh deploy usually fixes itself — otherwise commit the folder |
| `static/style.css missing` | `static/` not uploaded | commit it too (the site works but looks unstyled) |
| `database writable … unable to open` | `DB_PATH` points at a folder that does not exist, or the disk is not mounted | create the disk, or remove `DB_PATH` |
| service will not start at all (502 / no open ports) | `app.py` is not at the repository root | set Render’s *Root Directory*, or move the files up |

Rule of thumb: the repository root must contain `app.py`, `templates/`,
`static/`, `requirements.txt`.

## Other hosts

- **Railway / Fly.io** — same files, same start command.
- **PythonAnywhere** — free tier works; use the WSGI setup and point it at
  `app:app`.
- **Your own VPS** — `gunicorn app:app --bind 0.0.0.0:8000` behind nginx.

## Feature notes

- The exam **type** is the exam's name (no separate exam-name field); the same
  type cannot be created twice for one class and academic year.
- Exams are labelled `Type · Class · Year` everywhere.
- Date of admission is typed by the user — nothing is auto-filled.
- Marks entry saves the whole sheet in one go; out-of-range values are ignored.
- Results show total, percentage, CBSE grade (A1…E), failed subjects, pass/fail
  and rank with ties.
- CSV export for the student list and the result sheet.
