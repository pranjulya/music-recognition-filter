# music-recognition-filter

A Jupyter/Colab notebook that practises **SQL-based filtering for music recommendations** on a small SQLite database. The name says "recognition", but the project does no audio processing or song recognition. It is a set of SQL exercises over a toy music-streaming schema (users, songs, listens, recommendations). At the end it builds a simple "users who listened to X also listened to Y" recommendation query.

## What the notebook does

1. Installs `pysqlite3` and prints the SQLite version.
2. Defines two helpers:
   - `runSql(caption, query)` opens `music_streaming_1.db`, runs one query, and closes the connection.
   - `printSqlResults(cursor, tbl)` shows the result as a pandas DataFrame rendered as HTML.
3. Creates four tables (`CREATE TABLE IF NOT EXISTS`) in `music_streaming_1.db`.
4. Deletes any existing rows and inserts sample data: 4 users (Mickey, Minnie, Daffy, Pluto), 10 songs (Taylor Swift, Ed Sheeran, Beatles, plus one `DJ Mix` with a `NULL` genre) and 9 listens. Some listens have `NULL` ratings or listen times. No rows are inserted into `Recommendations`.
5. Runs a series of queries against that data (see below).
6. Ends with a comment-only cell listing follow-up assignment tasks: saving recommendations into the `Recommendations` table, generating recommendations for Minnie, redoing them based on listen time, and comparing the results. The notebook does not implement these tasks.

## Database schema (as created in the notebook)

| Table | Columns |
|---|---|
| `Users` | `user_id INTEGER PRIMARY KEY`, `name VARCHAR(100) NOT NULL`, `email VARCHAR(100) NOT NULL UNIQUE` |
| `Songs` | `song_id INTEGER PRIMARY KEY`, `title VARCHAR(100) NOT NULL`, `artist VARCHAR(100) NOT NULL`, `genre VARCHAR(100)` |
| `Listens` | `listen_id INTEGER PRIMARY KEY`, `user_id INTEGER NOT NULL` → `Users`, `song_id INTEGER NOT NULL` → `Songs`, `rating FLOAT`, `listen_time TIMESTAMP` |
| `Recommendations` | `recommendation_id INTEGER NOT NULL`, `recommendation_time TIMESTAMP`, `user_id INTEGER NOT NULL` → `Users`, `song_id INTEGER NOT NULL` → `Songs` |

## Kinds of queries run

- Filtering with `WHERE`, `LIKE`, and `IN` (for example, Classic songs, titles starting with "Ye", songs by Ed Sheeran or Taylor Swift)
- `SELECT` vs `SELECT DISTINCT` on genres
- `GROUP BY` with `COUNT(*)` (songs per artist and genre)
- `LEFT JOIN` across `Songs`, `Listens`, and `Users` to build one wide table
- Joins with filters and aggregates: songs rated above 4.6, average rating per song, most-listened songs (`ORDER BY COUNT(...) DESC`)
- `UNION` (Pop songs and Rock songs)
- A subquery with `IN (SELECT ...)`
- A recommendation query using CTEs (`WITH`) and a self-join on `Listens`

Examples copied from the notebook:

```sql
-- Number of songs by all artists in different genres
SELECT artist, genre,count(*) as num_songs
FROM Songs
GROUP BY artist, genre;
```

```sql
-- Popular songs by counting the listens
SELECT Songs.song_id, Songs.title, Songs.artist, count(Listens.song_id)
FROM Songs
JOIN Listens
ON Songs.song_id=Listens.song_id
GROUP BY Songs.title, Songs.artist
ORDER BY COUNT(Listens.song_id) DESC;
```

```sql
-- Song pairs shared across > 1 user, then recommend the paired song
-- to users who have listened to song1 but not song2
WITH song_similarity AS (
SELECT u1.song_id as song1, u2.song_id as song2, COUNT(*) AS common_users
FROM LISTENS u1
JOIN LISTENS u2
ON u1.user_id=u2.user_id
AND u1.song_id != u2.song_id
GROUP BY u1.song_id, u2.song_id
HAVING COUNT(*)>1
),

recs AS (
  SELECT user_id, song2 as song_id
  FROM song_similarity
  JOIN Listens as L
  ON L.song_id = song_similarity.song1
  WHERE song_similarity.song2 NOT IN
  (SELECT song_id FROM Listens as temp where temp.user_id=L.user_id)
)
select * from recs;
```

## Requirements

- Python 3. The notebook metadata only says "Python 3". The saved install output shows a `cp310` wheel, so it was last run on Python 3.10 in Colab.
- Packages imported:
  - `pysqlite3` (installed in the first cell with `!pip install pysqlite3`, used as `from pysqlite3 import dbapi2 as sqlite3`)
  - `pandas`
  - `IPython` (`IPython.display.display`, `HTML`)
- Jupyter (Notebook or Lab), VS Code with the Jupyter extension, or Google Colab.

`pysqlite3` is built from source when installed, so it may need a C compiler and SQLite headers outside Colab. The notebook only uses `connect`, `enable_callback_tracebacks`, and `sqlite_version_info`. Python's built-in `sqlite3` module also has all three, so you can replace the import with `import sqlite3` if the install fails.

## How to run

The notebook creates `music_streaming_1.db` in the current working directory. No external data is needed.

**Google Colab:** go to File → Open notebook → GitHub (or upload the file) and open `music_recommedation.ipynb`. Then use Runtime → Run all.

**Jupyter (local):**

```bash
git clone https://github.com/pranjulya/music-recognition-filter.git
cd music-recognition-filter
pip install jupyter pandas pysqlite3
jupyter notebook music_recommedation.ipynb
```

Run the cells from top to bottom. The first cell runs `!pip install pysqlite3` itself.

**VS Code:** open the folder, open `music_recommedation.ipynb`, select a Python 3 kernel that has `pandas` installed, and click "Run All".

Each run deletes and re-inserts the sample rows, so the notebook can be re-run safely.

## Notes

- The comment on the "songs listened to by user_id 1" query does not match the SQL. The query actually selects songs whose listens have a `NULL` `listen_time`.
- The second `runSql` call in the per-artist count cell reuses the caption "Count of songs by Taylor Swift in different genres".
- The final assignment cell's saved output shows a `SyntaxError` from an earlier run. Its current source is comments only.

## Project structure

```
music-recognition-filter/
├── README.md
└── music_recommedation.ipynb   # SQLite schema, sample data, and SQL queries
```

`music_streaming_1.db` is created when you run the notebook. It is not committed.
