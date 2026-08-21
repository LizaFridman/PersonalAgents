# Cards + Rules + EDHREC Synergy Cache Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build sub-project 1 of `mtg-agent`: a local SQLite card/rules database ingested from Scryfall's bulk data, deterministic deck-legality checking, a cached on-demand EDHREC synergy client, and an MCP server exposing all of it — runnable locally via Claude Code/Desktop.

**Architecture:** SQLAlchemy Core tables (`db/schema`) + Alembic migrations define the schema. `ingestion/scryfall` downloads and upserts Scryfall's bulk `oracle_cards`/`rulings` files. `ingestion/edhrec` is a thin read-through cache client, not a batch job. `modules/cards`, `modules/rules`, `modules/metagame` are pure query/logic layers over the DB. `apps/mcp-server` exposes five tools via the official Python MCP SDK's `FastMCP`.

**Tech Stack:** Python 3.11+, SQLAlchemy 2.x (Core, not ORM), Alembic, httpx, pytest, `mcp` (official Python MCP SDK).

**Spec:** `docs/superpowers/specs/2026-08-22-cards-rules-edhrec-design.md`

## Global Constraints

- SQLite only in this plan; no SQLite-only column types or functions — everything must be portable to Postgres later (per spec's Data model section).
- No live network calls in any test — all HTTP interactions are mocked/monkeypatched against fixture data captured from real, verified response shapes.
- `ingestion/edhrec` must never silently return stale-beyond-TTL or fabricated data on failure — raise/return an explicit error (per spec's "say plainly when data is missing" philosophy).
- Scryfall's card object fields used here (`id`, `oracle_id`, `name`, `mana_cost`, `cmc`, `colors`, `color_identity`, `type_line`, `oracle_text`, `power`, `toughness`, `loyalty`, `set`, `collector_number`, `rarity`, `legalities`, `prices`) and bulk-data object fields (`type`, `download_uri`, `updated_at`) are Scryfall's long-stable, long-documented public API surface (also already relied on by `mtg-commander-agent/project-instructions.md`'s `prices.usd`/`usd_foil`/`eur`/`tix` reference). The EDHREC `container.json_dict.cardlists` shape was fetched and verified live this session (see spec). Fixtures below are built from these; if the executor's environment allows a live check before Task 4/Task 9, spot-verify against `https://api.scryfall.com/cards/named?fuzzy=sol+ring` and `https://json.edhrec.com/pages/commanders/atraxa-praetors-voice.json` — this is a nice-to-have sanity check, not a blocker, since all tests run against fixtures either way.

---

## Task 1: Project scaffolding

**Files:**
- Create: `mtg-agent/pyproject.toml`
- Create: `mtg-agent/.gitignore`
- Create: `mtg-agent/modules/__init__.py`, `mtg-agent/modules/cards/__init__.py`, `mtg-agent/modules/rules/__init__.py`, `mtg-agent/modules/metagame/__init__.py`
- Create: `mtg-agent/ingestion/__init__.py`, `mtg-agent/ingestion/scryfall/__init__.py`, `mtg-agent/ingestion/edhrec/__init__.py`
- Create: `mtg-agent/db/__init__.py`, `mtg-agent/db/schema/__init__.py`
- Create: `mtg-agent/apps/__init__.py`, `mtg-agent/apps/mcp_server/__init__.py`
- Create: `mtg-agent/evaluations/__init__.py`, `mtg-agent/evaluations/legality/__init__.py`
- Create: `mtg-agent/tests/__init__.py`, `mtg-agent/tests/ingestion/__init__.py`, `mtg-agent/tests/modules/__init__.py`, `mtg-agent/tests/apps/__init__.py`

**Interfaces:**
- Produces: an installable package named `mtg-agent` with top-level importable packages `modules`, `ingestion`, `db`, `apps.mcp_server`, `evaluations`.

Note: `apps/mcp-server` in the spec's directory name uses a hyphen (not import-safe); the Python package underneath is `apps/mcp_server` — the directory on disk is `apps/mcp_server`, matching the spec's *intent* (an app under `apps/`) while staying a valid import path. Document this naming note in Task 14's `CLAUDE.md`.

- [ ] **Step 1: Create `pyproject.toml`**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mtg-agent"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "sqlalchemy>=2.0,<3.0",
    "alembic>=1.13,<2.0",
    "httpx>=0.27,<1.0",
    "mcp>=1.0,<2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0,<9.0",
    "pytest-mock>=3.14,<4.0",
]

[tool.hatch.build.targets.wheel]
packages = ["modules", "ingestion", "db", "apps", "evaluations"]
```

- [ ] **Step 2: Create `.gitignore`**

```
__pycache__/
*.pyc
.pytest_cache/
data/
*.sqlite
*.sqlite-journal
.venv/
*.egg-info/
```

- [ ] **Step 3: Create all `__init__.py` files listed above, each empty**

- [ ] **Step 4: Install the package in editable mode with dev extras**

Run (from `mtg-agent/`): `pip install -e ".[dev]"`
Expected: installs without error; `python -c "import modules, ingestion, db, apps.mcp_server, evaluations"` exits 0.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml .gitignore modules ingestion db apps evaluations tests
git commit -m "chore: scaffold mtg-agent package structure"
```

---

## Task 2: DB schema (SQLAlchemy Core tables) + engine helper

**Files:**
- Create: `mtg-agent/db/schema/models.py`
- Create: `mtg-agent/db/session.py`
- Test: `mtg-agent/tests/test_db_schema.py`

**Interfaces:**
- Produces: `db.schema.models.metadata` (a `sqlalchemy.MetaData`), and Table objects `cards`, `card_legalities`, `rulings`, `edhrec_synergy_cache`, `ingestion_runs` — all attached to `metadata`.
- Produces: `db.session.get_engine(db_path: str | None = None) -> sqlalchemy.engine.Engine`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/test_db_schema.py
from sqlalchemy import inspect
from db.schema.models import metadata
from db.session import get_engine


def test_create_all_tables():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    table_names = set(inspect(engine).get_table_names())
    assert table_names == {
        "cards",
        "card_legalities",
        "rulings",
        "edhrec_synergy_cache",
        "ingestion_runs",
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_db_schema.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'db.schema.models'`

- [ ] **Step 3: Write `db/schema/models.py`**

```python
from sqlalchemy import (
    Column,
    DateTime,
    Float,
    ForeignKey,
    Integer,
    JSON,
    MetaData,
    String,
    Table,
    Text,
)

metadata = MetaData()

cards = Table(
    "cards",
    metadata,
    Column("scryfall_id", String, primary_key=True),
    Column("oracle_id", String, nullable=False),
    Column("name", String, nullable=False),
    Column("mana_cost", String),
    Column("cmc", Float),
    Column("colors", String),
    Column("color_identity", String),
    Column("type_line", String),
    Column("oracle_text", Text),
    Column("power", String),
    Column("toughness", String),
    Column("loyalty", String),
    Column("set_code", String),
    Column("collector_number", String),
    Column("rarity", String),
    Column("price_usd", Float),
    Column("price_usd_foil", Float),
    Column("price_eur", Float),
    Column("price_tix", Float),
    Column("raw_json", JSON, nullable=False),
)

card_legalities = Table(
    "card_legalities",
    metadata,
    Column("card_id", String, ForeignKey("cards.scryfall_id"), primary_key=True),
    Column("format", String, primary_key=True),
    Column("status", String, nullable=False),
)

rulings = Table(
    "rulings",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("oracle_id", String, nullable=False),
    Column("published_at", String),
    Column("source", String),
    Column("comment", Text, nullable=False),
)

edhrec_synergy_cache = Table(
    "edhrec_synergy_cache",
    metadata,
    Column("commander_slug", String, primary_key=True),
    Column("cardlists_json", JSON, nullable=False),
    Column("fetched_at", DateTime, nullable=False),
)

ingestion_runs = Table(
    "ingestion_runs",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("source", String, nullable=False),
    Column("started_at", DateTime, nullable=False),
    Column("finished_at", DateTime),
    Column("record_count", Integer),
    Column("status", String, nullable=False),
)
```

- [ ] **Step 4: Write `db/session.py`**

```python
import os
from pathlib import Path

from sqlalchemy import create_engine
from sqlalchemy.engine import Engine

DEFAULT_DB_PATH = Path(__file__).resolve().parent.parent / "data" / "mtg.sqlite"


def get_engine(db_path: str | None = None) -> Engine:
    path = db_path or os.environ.get("MTG_DB_PATH") or str(DEFAULT_DB_PATH)
    if path != ":memory:":
        Path(path).parent.mkdir(parents=True, exist_ok=True)
    return create_engine(f"sqlite:///{path}")
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pytest tests/test_db_schema.py -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add db tests/test_db_schema.py
git commit -m "feat: add SQLAlchemy Core schema and engine helper"
```

---

## Task 3: Alembic migrations

**Files:**
- Create: `mtg-agent/alembic.ini`
- Create: `mtg-agent/db/migrations/env.py`
- Create: `mtg-agent/db/migrations/script.py.mako`
- Create: `mtg-agent/db/migrations/versions/0001_initial.py`
- Test: `mtg-agent/tests/test_migrations.py`

**Interfaces:**
- Consumes: `db.schema.models.metadata` (Task 2).
- Produces: a runnable Alembic environment (`alembic upgrade head` against any DB URL) that creates the same five tables as `metadata.create_all`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/test_migrations.py
import subprocess
import sys
from pathlib import Path

from sqlalchemy import create_engine, inspect

REPO_ROOT = Path(__file__).resolve().parent.parent


def test_alembic_upgrade_head_creates_tables(tmp_path):
    db_path = tmp_path / "migrated.sqlite"
    db_url = f"sqlite:///{db_path}"
    result = subprocess.run(
        [sys.executable, "-m", "alembic", "-x", f"db_url={db_url}", "upgrade", "head"],
        cwd=REPO_ROOT,
        capture_output=True,
        text=True,
    )
    assert result.returncode == 0, result.stderr

    engine = create_engine(db_url)
    table_names = set(inspect(engine).get_table_names())
    assert table_names == {
        "cards",
        "card_legalities",
        "rulings",
        "edhrec_synergy_cache",
        "ingestion_runs",
        "alembic_version",
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_migrations.py -v`
Expected: FAIL — `alembic.ini` / migration environment doesn't exist yet.

- [ ] **Step 3: Write `alembic.ini`**

```ini
[alembic]
script_location = db/migrations
sqlalchemy.url = sqlite:///data/mtg.sqlite

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
```

- [ ] **Step 4: Write `db/migrations/env.py`**

```python
from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from db.schema.models import metadata

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = metadata


def _db_url() -> str:
    x_args = context.get_x_argument(as_dictionary=True)
    return x_args.get("db_url") or config.get_main_option("sqlalchemy.url")


def run_migrations_offline() -> None:
    context.configure(
        url=_db_url(),
        target_metadata=target_metadata,
        literal_binds=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = _db_url()
    connectable = engine_from_config(
        configuration, prefix="sqlalchemy.", poolclass=pool.NullPool
    )
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

- [ ] **Step 5: Write `db/migrations/script.py.mako`**

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}
branch_labels = ${repr(branch_labels)}
depends_on = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

- [ ] **Step 6: Write `db/migrations/versions/0001_initial.py`**

```python
"""initial schema

Revision ID: 0001
Revises:
Create Date: 2026-08-22

"""
from alembic import op
import sqlalchemy as sa

revision = "0001"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "cards",
        sa.Column("scryfall_id", sa.String, primary_key=True),
        sa.Column("oracle_id", sa.String, nullable=False),
        sa.Column("name", sa.String, nullable=False),
        sa.Column("mana_cost", sa.String),
        sa.Column("cmc", sa.Float),
        sa.Column("colors", sa.String),
        sa.Column("color_identity", sa.String),
        sa.Column("type_line", sa.String),
        sa.Column("oracle_text", sa.Text),
        sa.Column("power", sa.String),
        sa.Column("toughness", sa.String),
        sa.Column("loyalty", sa.String),
        sa.Column("set_code", sa.String),
        sa.Column("collector_number", sa.String),
        sa.Column("rarity", sa.String),
        sa.Column("price_usd", sa.Float),
        sa.Column("price_usd_foil", sa.Float),
        sa.Column("price_eur", sa.Float),
        sa.Column("price_tix", sa.Float),
        sa.Column("raw_json", sa.JSON, nullable=False),
    )
    op.create_table(
        "card_legalities",
        sa.Column("card_id", sa.String, sa.ForeignKey("cards.scryfall_id"), primary_key=True),
        sa.Column("format", sa.String, primary_key=True),
        sa.Column("status", sa.String, nullable=False),
    )
    op.create_table(
        "rulings",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("oracle_id", sa.String, nullable=False),
        sa.Column("published_at", sa.String),
        sa.Column("source", sa.String),
        sa.Column("comment", sa.Text, nullable=False),
    )
    op.create_table(
        "edhrec_synergy_cache",
        sa.Column("commander_slug", sa.String, primary_key=True),
        sa.Column("cardlists_json", sa.JSON, nullable=False),
        sa.Column("fetched_at", sa.DateTime, nullable=False),
    )
    op.create_table(
        "ingestion_runs",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("source", sa.String, nullable=False),
        sa.Column("started_at", sa.DateTime, nullable=False),
        sa.Column("finished_at", sa.DateTime),
        sa.Column("record_count", sa.Integer),
        sa.Column("status", sa.String, nullable=False),
    )


def downgrade() -> None:
    op.drop_table("ingestion_runs")
    op.drop_table("edhrec_synergy_cache")
    op.drop_table("rulings")
    op.drop_table("card_legalities")
    op.drop_table("cards")
```

- [ ] **Step 7: Run test to verify it passes**

Run: `pytest tests/test_migrations.py -v`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add alembic.ini db/migrations tests/test_migrations.py
git commit -m "feat: add Alembic migrations for initial schema"
```

---

## Task 4: Scryfall card/ruling parsing

**Files:**
- Create: `mtg-agent/ingestion/scryfall/parse.py`
- Test: `mtg-agent/tests/ingestion/test_scryfall_parse.py`

**Interfaces:**
- Produces: `ingestion.scryfall.parse.parse_card(raw: dict) -> tuple[dict, list[dict]]` — returns `(card_row, legality_rows)`.
- Produces: `ingestion.scryfall.parse.parse_ruling(raw: dict) -> dict` — returns a `rulings` row.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/ingestion/test_scryfall_parse.py
from ingestion.scryfall.parse import parse_card, parse_ruling

SOL_RING_FIXTURE = {
    "id": "5d0b3c22-0000-0000-0000-000000000000",
    "oracle_id": "6c5e8592-0000-0000-0000-000000000000",
    "name": "Sol Ring",
    "mana_cost": "{1}",
    "cmc": 1.0,
    "colors": [],
    "color_identity": [],
    "type_line": "Artifact",
    "oracle_text": "{T}: Add {C}{C}.",
    "power": None,
    "toughness": None,
    "loyalty": None,
    "set": "cmr",
    "collector_number": "483",
    "rarity": "uncommon",
    "legalities": {
        "commander": "legal",
        "modern": "not_legal",
        "vintage": "restricted",
    },
    "prices": {"usd": "1.85", "usd_foil": "12.00", "eur": None, "tix": None},
}

RULING_FIXTURE = {
    "oracle_id": "6c5e8592-0000-0000-0000-000000000000",
    "source": "wotc",
    "published_at": "2004-10-04",
    "comment": "This card doesn't tap for colored mana.",
}


def test_parse_card_splits_card_row_and_legalities():
    card_row, legality_rows = parse_card(SOL_RING_FIXTURE)

    assert card_row["scryfall_id"] == "5d0b3c22-0000-0000-0000-000000000000"
    assert card_row["oracle_id"] == "6c5e8592-0000-0000-0000-000000000000"
    assert card_row["name"] == "Sol Ring"
    assert card_row["colors"] == ""
    assert card_row["color_identity"] == ""
    assert card_row["set_code"] == "cmr"
    assert card_row["price_usd"] == 1.85
    assert card_row["price_usd_foil"] == 12.00
    assert card_row["price_eur"] is None
    assert card_row["raw_json"] == SOL_RING_FIXTURE

    assert {"card_id": card_row["scryfall_id"], "format": "commander", "status": "legal"} in legality_rows
    assert {"card_id": card_row["scryfall_id"], "format": "vintage", "status": "restricted"} in legality_rows
    assert len(legality_rows) == 3


def test_parse_card_joins_multiple_colors():
    fixture = dict(SOL_RING_FIXTURE, colors=["U", "B"], color_identity=["U", "B"])
    card_row, _ = parse_card(fixture)
    assert card_row["colors"] == "U,B"
    assert card_row["color_identity"] == "U,B"


def test_parse_ruling():
    row = parse_ruling(RULING_FIXTURE)
    assert row == {
        "oracle_id": "6c5e8592-0000-0000-0000-000000000000",
        "published_at": "2004-10-04",
        "source": "wotc",
        "comment": "This card doesn't tap for colored mana.",
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/ingestion/test_scryfall_parse.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'ingestion.scryfall.parse'`

- [ ] **Step 3: Write `ingestion/scryfall/parse.py`**

```python
def parse_card(raw: dict) -> tuple[dict, list[dict]]:
    prices = raw.get("prices") or {}

    def _price(key: str) -> float | None:
        value = prices.get(key)
        return float(value) if value is not None else None

    card_row = {
        "scryfall_id": raw["id"],
        "oracle_id": raw["oracle_id"],
        "name": raw["name"],
        "mana_cost": raw.get("mana_cost"),
        "cmc": raw.get("cmc"),
        "colors": ",".join(raw.get("colors") or []),
        "color_identity": ",".join(raw.get("color_identity") or []),
        "type_line": raw.get("type_line"),
        "oracle_text": raw.get("oracle_text"),
        "power": raw.get("power"),
        "toughness": raw.get("toughness"),
        "loyalty": raw.get("loyalty"),
        "set_code": raw.get("set"),
        "collector_number": raw.get("collector_number"),
        "rarity": raw.get("rarity"),
        "price_usd": _price("usd"),
        "price_usd_foil": _price("usd_foil"),
        "price_eur": _price("eur"),
        "price_tix": _price("tix"),
        "raw_json": raw,
    }

    legality_rows = [
        {"card_id": card_row["scryfall_id"], "format": fmt, "status": status}
        for fmt, status in (raw.get("legalities") or {}).items()
    ]

    return card_row, legality_rows


def parse_ruling(raw: dict) -> dict:
    return {
        "oracle_id": raw["oracle_id"],
        "published_at": raw.get("published_at"),
        "source": raw.get("source"),
        "comment": raw["comment"],
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/ingestion/test_scryfall_parse.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ingestion/scryfall/parse.py tests/ingestion/test_scryfall_parse.py
git commit -m "feat: parse Scryfall card and ruling objects into DB rows"
```

---

## Task 5: Scryfall bulk-data URL client

**Files:**
- Create: `mtg-agent/ingestion/scryfall/client.py`
- Test: `mtg-agent/tests/ingestion/test_scryfall_client.py`

**Interfaces:**
- Produces: `ingestion.scryfall.client.get_bulk_data_url(bulk_type: str, http_client: httpx.Client) -> str` — raises `ValueError` if `bulk_type` isn't found in the bulk-data index.
- Produces: `ingestion.scryfall.client.download_bulk_data(url: str, http_client: httpx.Client) -> list[dict]` — GETs the URL, returns parsed JSON array.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/ingestion/test_scryfall_client.py
import httpx
import pytest

from ingestion.scryfall.client import download_bulk_data, get_bulk_data_url

BULK_DATA_INDEX = {
    "object": "list",
    "data": [
        {
            "object": "bulk_data",
            "id": "aaaa",
            "type": "oracle_cards",
            "updated_at": "2026-08-22T00:00:00.000+00:00",
            "uri": "https://api.scryfall.com/bulk-data/aaaa",
            "name": "Oracle Cards",
            "download_uri": "https://data.scryfall.io/oracle-cards/oracle-cards.json",
            "content_type": "application/json",
            "content_encoding": "gzip",
        },
        {
            "object": "bulk_data",
            "id": "bbbb",
            "type": "rulings",
            "updated_at": "2026-08-22T00:00:00.000+00:00",
            "uri": "https://api.scryfall.com/bulk-data/bbbb",
            "name": "Rulings",
            "download_uri": "https://data.scryfall.io/rulings/rulings.json",
            "content_type": "application/json",
            "content_encoding": "gzip",
        },
    ],
}


def _client_with_response(url_to_json: dict[str, object]) -> httpx.Client:
    def handler(request: httpx.Request) -> httpx.Response:
        return httpx.Response(200, json=url_to_json[str(request.url)])

    return httpx.Client(transport=httpx.MockTransport(handler))


def test_get_bulk_data_url_finds_oracle_cards():
    client = _client_with_response({"https://api.scryfall.com/bulk-data": BULK_DATA_INDEX})
    url = get_bulk_data_url("oracle_cards", client)
    assert url == "https://data.scryfall.io/oracle-cards/oracle-cards.json"


def test_get_bulk_data_url_missing_type_raises():
    client = _client_with_response({"https://api.scryfall.com/bulk-data": BULK_DATA_INDEX})
    with pytest.raises(ValueError, match="not_a_real_type"):
        get_bulk_data_url("not_a_real_type", client)


def test_download_bulk_data_returns_parsed_json():
    cards = [{"id": "1", "name": "Sol Ring"}]
    client = _client_with_response({"https://data.scryfall.io/oracle-cards/oracle-cards.json": cards})
    result = download_bulk_data("https://data.scryfall.io/oracle-cards/oracle-cards.json", client)
    assert result == cards
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/ingestion/test_scryfall_client.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'ingestion.scryfall.client'`

- [ ] **Step 3: Write `ingestion/scryfall/client.py`**

```python
import httpx

BULK_DATA_INDEX_URL = "https://api.scryfall.com/bulk-data"
USER_AGENT = "mtg-agent/0.1 (personal Commander assistant; local ingestion script)"


def get_bulk_data_url(bulk_type: str, http_client: httpx.Client) -> str:
    response = http_client.get(
        BULK_DATA_INDEX_URL, headers={"User-Agent": USER_AGENT, "Accept": "application/json"}
    )
    response.raise_for_status()
    for entry in response.json()["data"]:
        if entry["type"] == bulk_type:
            return entry["download_uri"]
    raise ValueError(f"No bulk-data entry found for type={bulk_type!r}")


def download_bulk_data(url: str, http_client: httpx.Client) -> list[dict]:
    response = http_client.get(url, headers={"User-Agent": USER_AGENT, "Accept": "application/json"})
    response.raise_for_status()
    return response.json()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/ingestion/test_scryfall_client.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ingestion/scryfall/client.py tests/ingestion/test_scryfall_client.py
git commit -m "feat: add Scryfall bulk-data index and download client"
```

---

## Task 6: Scryfall ingestion run (upsert + logging)

**Files:**
- Create: `mtg-agent/ingestion/scryfall/run.py`
- Test: `mtg-agent/tests/ingestion/test_scryfall_run.py`

**Interfaces:**
- Consumes: `ingestion.scryfall.client.get_bulk_data_url`, `download_bulk_data` (Task 5); `ingestion.scryfall.parse.parse_card`, `parse_ruling` (Task 4); `db.schema.models` tables, `db.session.get_engine` (Task 2).
- Produces: `ingestion.scryfall.run.ingest(engine, http_client: httpx.Client) -> dict` — runs both bulk types, upserts rows, writes two `ingestion_runs` rows, returns `{"cards": n, "rulings": m}`.
- Produces: `ingestion.scryfall.run.main()` — CLI entry point (`python -m ingestion.scryfall.run`), builds a real engine/http client and calls `ingest`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/ingestion/test_scryfall_run.py
import httpx
from sqlalchemy import select

from db.schema.models import cards, card_legalities, ingestion_runs, metadata, rulings
from db.session import get_engine
from ingestion.scryfall.run import ingest

BULK_INDEX = {
    "object": "list",
    "data": [
        {"type": "oracle_cards", "download_uri": "https://data.scryfall.io/oracle-cards.json"},
        {"type": "rulings", "download_uri": "https://data.scryfall.io/rulings.json"},
    ],
}

CARD_DATA = [
    {
        "id": "card-1",
        "oracle_id": "oracle-1",
        "name": "Sol Ring",
        "mana_cost": "{1}",
        "cmc": 1.0,
        "colors": [],
        "color_identity": [],
        "type_line": "Artifact",
        "oracle_text": "{T}: Add {C}{C}.",
        "power": None,
        "toughness": None,
        "loyalty": None,
        "set": "cmr",
        "collector_number": "483",
        "rarity": "uncommon",
        "legalities": {"commander": "legal"},
        "prices": {"usd": "1.85", "usd_foil": None, "eur": None, "tix": None},
    }
]

RULING_DATA = [
    {"oracle_id": "oracle-1", "source": "wotc", "published_at": "2004-10-04", "comment": "Doesn't tap for colored mana."}
]


def _mock_client() -> httpx.Client:
    responses = {
        "https://api.scryfall.com/bulk-data": BULK_INDEX,
        "https://data.scryfall.io/oracle-cards.json": CARD_DATA,
        "https://data.scryfall.io/rulings.json": RULING_DATA,
    }

    def handler(request: httpx.Request) -> httpx.Response:
        return httpx.Response(200, json=responses[str(request.url)])

    return httpx.Client(transport=httpx.MockTransport(handler))


def test_ingest_populates_all_tables_and_logs_runs():
    engine = get_engine(":memory:")
    metadata.create_all(engine)

    result = ingest(engine, _mock_client())

    assert result == {"cards": 1, "rulings": 1}

    with engine.connect() as conn:
        card_rows = conn.execute(select(cards)).fetchall()
        assert len(card_rows) == 1
        assert card_rows[0].name == "Sol Ring"

        legality_rows = conn.execute(select(card_legalities)).fetchall()
        assert len(legality_rows) == 1
        assert legality_rows[0].status == "legal"

        ruling_rows = conn.execute(select(rulings)).fetchall()
        assert len(ruling_rows) == 1

        run_rows = conn.execute(select(ingestion_runs)).fetchall()
        assert {r.source for r in run_rows} == {"scryfall_cards", "scryfall_rulings"}
        assert all(r.status == "success" for r in run_rows)
        assert all(r.finished_at is not None for r in run_rows)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/ingestion/test_scryfall_run.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'ingestion.scryfall.run'`

- [ ] **Step 3: Write `ingestion/scryfall/run.py`**

```python
from datetime import datetime, timezone

import httpx
from sqlalchemy import delete
from sqlalchemy.engine import Engine

from db.schema.models import card_legalities, cards, ingestion_runs, rulings
from db.session import get_engine
from ingestion.scryfall.client import download_bulk_data, get_bulk_data_url
from ingestion.scryfall.parse import parse_card, parse_ruling


def _log_run(conn, source: str, started_at: datetime, record_count: int, status: str) -> None:
    conn.execute(
        ingestion_runs.insert().values(
            source=source,
            started_at=started_at,
            finished_at=datetime.now(timezone.utc),
            record_count=record_count,
            status=status,
        )
    )


def _ingest_cards(engine: Engine, http_client: httpx.Client) -> int:
    started_at = datetime.now(timezone.utc)
    url = get_bulk_data_url("oracle_cards", http_client)
    raw_cards = download_bulk_data(url, http_client)

    card_rows = []
    legality_rows = []
    for raw in raw_cards:
        card_row, legalities = parse_card(raw)
        card_rows.append(card_row)
        legality_rows.extend(legalities)

    with engine.begin() as conn:
        conn.execute(delete(card_legalities))
        conn.execute(delete(cards))
        if card_rows:
            conn.execute(cards.insert(), card_rows)
        if legality_rows:
            conn.execute(card_legalities.insert(), legality_rows)
        _log_run(conn, "scryfall_cards", started_at, len(card_rows), "success")

    return len(card_rows)


def _ingest_rulings(engine: Engine, http_client: httpx.Client) -> int:
    started_at = datetime.now(timezone.utc)
    url = get_bulk_data_url("rulings", http_client)
    raw_rulings = download_bulk_data(url, http_client)
    ruling_rows = [parse_ruling(raw) for raw in raw_rulings]

    with engine.begin() as conn:
        conn.execute(delete(rulings))
        if ruling_rows:
            conn.execute(rulings.insert(), ruling_rows)
        _log_run(conn, "scryfall_rulings", started_at, len(ruling_rows), "success")

    return len(ruling_rows)


def ingest(engine: Engine, http_client: httpx.Client) -> dict:
    return {
        "cards": _ingest_cards(engine, http_client),
        "rulings": _ingest_rulings(engine, http_client),
    }


def main() -> None:
    engine = get_engine()
    with httpx.Client(timeout=30.0) as http_client:
        result = ingest(engine, http_client)
    print(f"Ingested {result['cards']} cards and {result['rulings']} rulings.")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/ingestion/test_scryfall_run.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ingestion/scryfall/run.py tests/ingestion/test_scryfall_run.py
git commit -m "feat: ingest Scryfall bulk cards and rulings into the DB"
```

---

## Task 7: `modules/cards` query functions

**Files:**
- Create: `mtg-agent/modules/cards/queries.py`
- Test: `mtg-agent/tests/modules/test_cards.py`

**Interfaces:**
- Consumes: `db.schema.models.cards`, `card_legalities` (Task 2).
- Produces: `modules.cards.queries.get_card_by_name(engine, name: str) -> dict | None` (case-insensitive exact match).
- Produces: `modules.cards.queries.search_cards(engine, *, color_identity_subset: list[str] | None = None, type_contains: str | None = None, legal_in: str | None = None) -> list[dict]`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/modules/test_cards.py
from sqlalchemy import select

from db.schema.models import card_legalities, cards, metadata
from db.session import get_engine
from modules.cards.queries import get_card_by_name, search_cards


def _seed(engine):
    metadata.create_all(engine)
    with engine.begin() as conn:
        conn.execute(
            cards.insert(),
            [
                dict(
                    scryfall_id="1", oracle_id="o1", name="Sol Ring", mana_cost="{1}", cmc=1.0,
                    colors="", color_identity="", type_line="Artifact", oracle_text="", power=None,
                    toughness=None, loyalty=None, set_code="cmr", collector_number="483", rarity="uncommon",
                    price_usd=1.85, price_usd_foil=None, price_eur=None, price_tix=None, raw_json={},
                ),
                dict(
                    scryfall_id="2", oracle_id="o2", name="Swords to Plowshares", mana_cost="{W}", cmc=1.0,
                    colors="W", color_identity="W", type_line="Instant", oracle_text="", power=None,
                    toughness=None, loyalty=None, set_code="stx", collector_number="34", rarity="uncommon",
                    price_usd=0.50, price_usd_foil=None, price_eur=None, price_tix=None, raw_json={},
                ),
            ],
        )
        conn.execute(
            card_legalities.insert(),
            [
                {"card_id": "1", "format": "commander", "status": "legal"},
                {"card_id": "2", "format": "commander", "status": "legal"},
            ],
        )
    return engine


def test_get_card_by_name_case_insensitive():
    engine = _seed(get_engine(":memory:"))
    card = get_card_by_name(engine, "sol ring")
    assert card["name"] == "Sol Ring"
    assert get_card_by_name(engine, "Nonexistent Card") is None


def test_search_cards_by_color_identity_subset():
    engine = _seed(get_engine(":memory:"))
    results = search_cards(engine, color_identity_subset=["W"])
    names = {c["name"] for c in results}
    assert names == {"Swords to Plowshares"}


def test_search_cards_by_type_and_legality():
    engine = _seed(get_engine(":memory:"))
    results = search_cards(engine, type_contains="Artifact", legal_in="commander")
    assert [c["name"] for c in results] == ["Sol Ring"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/modules/test_cards.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'modules.cards.queries'`

- [ ] **Step 3: Write `modules/cards/queries.py`**

```python
from sqlalchemy import select
from sqlalchemy.engine import Engine

from db.schema.models import card_legalities, cards


def _row_to_dict(row) -> dict:
    return dict(row._mapping)


def get_card_by_name(engine: Engine, name: str) -> dict | None:
    stmt = select(cards).where(cards.c.name.ilike(name))
    with engine.connect() as conn:
        row = conn.execute(stmt).first()
    return _row_to_dict(row) if row else None


def search_cards(
    engine: Engine,
    *,
    color_identity_subset: list[str] | None = None,
    type_contains: str | None = None,
    legal_in: str | None = None,
) -> list[dict]:
    stmt = select(cards)
    if type_contains:
        stmt = stmt.where(cards.c.type_line.ilike(f"%{type_contains}%"))
    if legal_in:
        stmt = stmt.join(card_legalities, card_legalities.c.card_id == cards.c.scryfall_id).where(
            card_legalities.c.format == legal_in, card_legalities.c.status == "legal"
        )

    with engine.connect() as conn:
        rows = [_row_to_dict(r) for r in conn.execute(stmt)]

    if color_identity_subset is not None:
        allowed = set(color_identity_subset)
        rows = [
            r
            for r in rows
            if set(c for c in r["color_identity"].split(",") if c) <= allowed
        ]

    return rows
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/modules/test_cards.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add modules/cards/queries.py tests/modules/test_cards.py
git commit -m "feat: add modules/cards query functions"
```

---

## Task 8: `modules/rules` deck-legality checks

**Files:**
- Create: `mtg-agent/modules/rules/legality.py`
- Test: `mtg-agent/tests/modules/test_rules.py`

**Interfaces:**
- Consumes: `db.schema.models.cards`, `card_legalities` (Task 2).
- Produces: `modules.rules.legality.check_deck_legality(engine, *, commander_names: list[str], deck_card_names: list[str], format: str = "commander") -> dict` — returns `{"legal": bool, "violations": list[str]}`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/modules/test_rules.py
from db.schema.models import card_legalities, cards, metadata
from db.session import get_engine
from modules.rules.legality import check_deck_legality


def _card(scryfall_id, name, color_identity, legal=True):
    return dict(
        scryfall_id=scryfall_id, oracle_id=f"o-{scryfall_id}", name=name, mana_cost="", cmc=0.0,
        colors="", color_identity=color_identity, type_line="", oracle_text="", power=None,
        toughness=None, loyalty=None, set_code="tst", collector_number="1", rarity="common",
        price_usd=None, price_usd_foil=None, price_eur=None, price_tix=None, raw_json={},
    ), {"card_id": scryfall_id, "format": "commander", "status": "legal" if legal else "banned"}


def _seed(engine, deck_size=100):
    metadata.create_all(engine)
    card_rows = []
    legality_rows = []

    commander_row, commander_legality = _card("cmd", "Teysa Karlov", "W,B")
    card_rows.append(commander_row)
    legality_rows.append(commander_legality)

    for i in range(deck_size - 1):
        row, legality = _card(f"c{i}", f"Card {i}", "W,B" if i != 0 else "U")
        card_rows.append(row)
        legality_rows.append(legality)

    banned_row, banned_legality = _card("banned1", "Banned Card", "W,B", legal=False)
    card_rows.append(banned_row)
    legality_rows.append(banned_legality)

    with engine.begin() as conn:
        conn.execute(cards.insert(), card_rows)
        conn.execute(card_legalities.insert(), legality_rows)
    return engine


def test_legal_deck_passes():
    engine = _seed(get_engine(":memory:"))
    deck_names = ["Teysa Karlov"] + [f"Card {i}" for i in range(99)]
    result = check_deck_legality(engine, commander_names=["Teysa Karlov"], deck_card_names=deck_names)
    assert result == {"legal": True, "violations": []}


def test_wrong_card_count_fails():
    engine = _seed(get_engine(":memory:"))
    deck_names = ["Teysa Karlov"] + [f"Card {i}" for i in range(50)]
    result = check_deck_legality(engine, commander_names=["Teysa Karlov"], deck_card_names=deck_names)
    assert result["legal"] is False
    assert any("100 cards" in v for v in result["violations"])


def test_non_singleton_fails():
    engine = _seed(get_engine(":memory:"))
    deck_names = ["Teysa Karlov"] + [f"Card {i}" for i in range(98)] + ["Card 0"]
    result = check_deck_legality(engine, commander_names=["Teysa Karlov"], deck_card_names=deck_names)
    assert result["legal"] is False
    assert any("Card 0" in v and "singleton" in v for v in result["violations"])


def test_color_identity_violation_fails():
    engine = _seed(get_engine(":memory:"))
    deck_names = ["Teysa Karlov"] + [f"Card {i}" for i in range(99)]
    result = check_deck_legality(engine, commander_names=["Teysa Karlov"], deck_card_names=deck_names)
    assert result["legal"] is False
    assert any("Card 0" in v and "color identity" in v for v in result["violations"])


def test_banned_card_fails():
    engine = _seed(get_engine(":memory:"))
    deck_names = ["Teysa Karlov"] + [f"Card {i}" for i in range(98)] + ["Banned Card"]
    result = check_deck_legality(engine, commander_names=["Teysa Karlov"], deck_card_names=deck_names)
    assert result["legal"] is False
    assert any("Banned Card" in v and "banned" in v for v in result["violations"])
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/modules/test_rules.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'modules.rules.legality'`

- [ ] **Step 3: Write `modules/rules/legality.py`**

```python
from collections import Counter

from sqlalchemy import select
from sqlalchemy.engine import Engine

from db.schema.models import card_legalities, cards


def _get_card_by_name(conn, name: str) -> dict | None:
    row = conn.execute(select(cards).where(cards.c.name == name)).first()
    return dict(row._mapping) if row else None


def _legality_status(conn, card_id: str, format: str) -> str | None:
    row = conn.execute(
        select(card_legalities.c.status).where(
            card_legalities.c.card_id == card_id, card_legalities.c.format == format
        )
    ).first()
    return row.status if row else None


def check_deck_legality(
    engine: Engine,
    *,
    commander_names: list[str],
    deck_card_names: list[str],
    format: str = "commander",
) -> dict:
    violations: list[str] = []

    with engine.connect() as conn:
        commander_rows = [_get_card_by_name(conn, name) for name in commander_names]
        for name, row in zip(commander_names, commander_rows):
            if row is None:
                violations.append(f"Commander '{name}' not found in card database.")

        commander_identity: set[str] = set()
        for row in commander_rows:
            if row:
                commander_identity |= {c for c in row["color_identity"].split(",") if c}

        total_count = len(deck_card_names) + len(commander_names)
        if total_count != 100:
            violations.append(f"Deck must be exactly 100 cards including commander(s); got {total_count}.")

        name_counts = Counter(deck_card_names)
        for name, count in name_counts.items():
            if count > 1:
                violations.append(f"'{name}' violates singleton (appears {count} times).")

        for name in deck_card_names:
            row = _get_card_by_name(conn, name)
            if row is None:
                violations.append(f"'{name}' not found in card database.")
                continue

            card_identity = {c for c in row["color_identity"].split(",") if c}
            if not card_identity <= commander_identity:
                violations.append(
                    f"'{name}' color identity {sorted(card_identity)} is not a subset of "
                    f"commander color identity {sorted(commander_identity)}."
                )

            status = _legality_status(conn, row["scryfall_id"], format)
            if status == "banned":
                violations.append(f"'{name}' is banned in {format}.")
            elif status == "not_legal":
                violations.append(f"'{name}' is not legal in {format}.")

    return {"legal": len(violations) == 0, "violations": violations}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/modules/test_rules.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add modules/rules/legality.py tests/modules/test_rules.py
git commit -m "feat: add deterministic deck-legality checking"
```

---

## Task 9: `evaluations/legality` fixed case suite

**Files:**
- Create: `mtg-agent/evaluations/legality/cases.py`
- Create: `mtg-agent/evaluations/legality/run_eval.py`
- Test: `mtg-agent/tests/test_evaluations_legality.py`

**Interfaces:**
- Consumes: `modules.rules.legality.check_deck_legality` (Task 8).
- Produces: `evaluations.legality.cases.CASES: list[dict]` — each `{"name": str, "commander_names": [...], "deck_card_names": [...], "expected_legal": bool}`.
- Produces: `evaluations.legality.run_eval.run(engine) -> list[dict]` — returns per-case `{"name", "passed", "result"}`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/test_evaluations_legality.py
from db.schema.models import card_legalities, cards, metadata
from db.session import get_engine
from evaluations.legality.run_eval import run


def _card(scryfall_id, name, color_identity, legal_status="legal"):
    return dict(
        scryfall_id=scryfall_id, oracle_id=f"o-{scryfall_id}", name=name, mana_cost="", cmc=0.0,
        colors="", color_identity=color_identity, type_line="", oracle_text="", power=None,
        toughness=None, loyalty=None, set_code="tst", collector_number="1", rarity="common",
        price_usd=None, price_usd_foil=None, price_eur=None, price_tix=None, raw_json={},
    ), {"card_id": scryfall_id, "format": "commander", "status": legal_status}


def _seed(engine):
    metadata.create_all(engine)
    rows, legalities = [], []

    cmd_row, cmd_leg = _card("cmd", "Teysa Karlov", "W,B")
    rows.append(cmd_row)
    legalities.append(cmd_leg)

    for i in range(99):
        row, leg = _card(f"c{i}", f"Card {i}", "W,B")
        rows.append(row)
        legalities.append(leg)

    off_row, off_leg = _card("off", "Off Color Card", "U")
    rows.append(off_row)
    legalities.append(off_leg)

    banned_row, banned_leg = _card("banned", "Banned Card", "W,B", legal_status="banned")
    rows.append(banned_row)
    legalities.append(banned_leg)

    with engine.begin() as conn:
        conn.execute(cards.insert(), rows)
        conn.execute(card_legalities.insert(), legalities)
    return engine


def test_all_eval_cases_pass():
    engine = _seed(get_engine(":memory:"))
    results = run(engine)
    failed = [r for r in results if not r["passed"]]
    assert failed == [], failed
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_evaluations_legality.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'evaluations.legality.run_eval'`

- [ ] **Step 3: Write `evaluations/legality/cases.py`**

```python
CASES = [
    {
        "name": "legal_100_card_deck",
        "commander_names": ["Teysa Karlov"],
        "deck_card_names": [f"Card {i}" for i in range(99)],
        "expected_legal": True,
    },
    {
        "name": "wrong_card_count",
        "commander_names": ["Teysa Karlov"],
        "deck_card_names": [f"Card {i}" for i in range(50)],
        "expected_legal": False,
    },
    {
        "name": "non_singleton",
        "commander_names": ["Teysa Karlov"],
        "deck_card_names": [f"Card {i}" for i in range(98)] + ["Card 0"],
        "expected_legal": False,
    },
    {
        "name": "off_color_card",
        "commander_names": ["Teysa Karlov"],
        "deck_card_names": [f"Card {i}" for i in range(98)] + ["Off Color Card"],
        "expected_legal": False,
    },
    {
        "name": "banned_card",
        "commander_names": ["Teysa Karlov"],
        "deck_card_names": [f"Card {i}" for i in range(98)] + ["Banned Card"],
        "expected_legal": False,
    },
]
```

- [ ] **Step 4: Write `evaluations/legality/run_eval.py`**

```python
from sqlalchemy.engine import Engine

from evaluations.legality.cases import CASES
from modules.rules.legality import check_deck_legality


def run(engine: Engine) -> list[dict]:
    results = []
    for case in CASES:
        result = check_deck_legality(
            engine,
            commander_names=case["commander_names"],
            deck_card_names=case["deck_card_names"],
        )
        results.append(
            {
                "name": case["name"],
                "passed": result["legal"] == case["expected_legal"],
                "result": result,
            }
        )
    return results


if __name__ == "__main__":
    from db.session import get_engine

    for r in run(get_engine()):
        status = "PASS" if r["passed"] else "FAIL"
        print(f"[{status}] {r['name']}")
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pytest tests/test_evaluations_legality.py -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add evaluations/legality tests/test_evaluations_legality.py
git commit -m "feat: add legality evaluation case suite"
```

---

## Task 10: EDHREC cached client

**Files:**
- Create: `mtg-agent/ingestion/edhrec/client.py`
- Test: `mtg-agent/tests/ingestion/test_edhrec_client.py`

**Interfaces:**
- Consumes: `db.schema.models.edhrec_synergy_cache` (Task 2).
- Produces: `ingestion.edhrec.client.slugify(commander_name: str) -> str`.
- Produces: `ingestion.edhrec.client.get_commander_synergy(engine, commander_name: str, http_client: httpx.Client, *, ttl_days: int = 7, now: datetime | None = None) -> list[dict]` — returns the `cardlists` list; raises `RuntimeError` on fetch/parse failure with no usable cache.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/ingestion/test_edhrec_client.py
from datetime import datetime, timedelta, timezone

import httpx
import pytest
from sqlalchemy import select

from db.schema.models import edhrec_synergy_cache, metadata
from db.session import get_engine
from ingestion.edhrec.client import get_commander_synergy, slugify

CARDLISTS = [
    {
        "header": "High Synergy Cards",
        "tag": "highsynergycards",
        "cardviews": [{"name": "Tekuthal, Inquiry Dominus", "synergy": 0.27, "num_decks": 29032}],
    }
]

EDHREC_RESPONSE = {"container": {"json_dict": {"cardlists": CARDLISTS}}}


def _mock_client(json_body=None, status_code=200):
    def handler(request: httpx.Request) -> httpx.Response:
        return httpx.Response(status_code, json=json_body or {})

    return httpx.Client(transport=httpx.MockTransport(handler))


def test_slugify():
    assert slugify("Atraxa, Praetors' Voice") == "atraxa-praetors-voice"
    assert slugify("Teysa Karlov") == "teysa-karlov"


def test_cache_miss_fetches_and_caches():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    client = _mock_client(EDHREC_RESPONSE)

    result = get_commander_synergy(engine, "Atraxa, Praetors' Voice", client)

    assert result == CARDLISTS
    with engine.connect() as conn:
        row = conn.execute(
            select(edhrec_synergy_cache).where(
                edhrec_synergy_cache.c.commander_slug == "atraxa-praetors-voice"
            )
        ).first()
    assert row is not None
    assert row.cardlists_json == CARDLISTS


def test_fresh_cache_hit_does_not_call_http():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    now = datetime.now(timezone.utc)

    with engine.begin() as conn:
        conn.execute(
            edhrec_synergy_cache.insert().values(
                commander_slug="teysa-karlov", cardlists_json=CARDLISTS, fetched_at=now
            )
        )

    def handler(request: httpx.Request) -> httpx.Response:
        raise AssertionError("HTTP should not be called on a fresh cache hit")

    client = httpx.Client(transport=httpx.MockTransport(handler))

    result = get_commander_synergy(engine, "Teysa Karlov", client, now=now)
    assert result == CARDLISTS


def test_stale_cache_refetches():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    old = datetime.now(timezone.utc) - timedelta(days=30)

    with engine.begin() as conn:
        conn.execute(
            edhrec_synergy_cache.insert().values(
                commander_slug="teysa-karlov", cardlists_json=[], fetched_at=old
            )
        )

    client = _mock_client(EDHREC_RESPONSE)
    result = get_commander_synergy(engine, "Teysa Karlov", client, now=datetime.now(timezone.utc))
    assert result == CARDLISTS


def test_fetch_failure_with_no_cache_raises():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    client = _mock_client(status_code=500)

    with pytest.raises(RuntimeError, match="Teysa Karlov"):
        get_commander_synergy(engine, "Teysa Karlov", client)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/ingestion/test_edhrec_client.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'ingestion.edhrec.client'`

- [ ] **Step 3: Write `ingestion/edhrec/client.py`**

```python
import re
from datetime import datetime, timedelta, timezone

import httpx
from sqlalchemy import select
from sqlalchemy.engine import Engine

from db.schema.models import edhrec_synergy_cache

USER_AGENT = "mtg-agent/0.1 (personal Commander assistant; interactive on-demand lookups)"


def slugify(commander_name: str) -> str:
    lowered = commander_name.lower()
    stripped = re.sub(r"[^a-z0-9\s-]", "", lowered)
    return re.sub(r"[\s]+", "-", stripped.strip())


def _cached_row(engine: Engine, slug: str) -> dict | None:
    with engine.connect() as conn:
        row = conn.execute(
            select(edhrec_synergy_cache).where(edhrec_synergy_cache.c.commander_slug == slug)
        ).first()
    return dict(row._mapping) if row else None


def _store(engine: Engine, slug: str, cardlists: list[dict], fetched_at: datetime) -> None:
    with engine.begin() as conn:
        conn.execute(edhrec_synergy_cache.delete().where(edhrec_synergy_cache.c.commander_slug == slug))
        conn.execute(
            edhrec_synergy_cache.insert().values(
                commander_slug=slug, cardlists_json=cardlists, fetched_at=fetched_at
            )
        )


def _fetch(slug: str, http_client: httpx.Client) -> list[dict]:
    url = f"https://json.edhrec.com/pages/commanders/{slug}.json"
    response = http_client.get(url, headers={"User-Agent": USER_AGENT, "Accept": "application/json"})
    response.raise_for_status()
    return response.json()["container"]["json_dict"]["cardlists"]


def get_commander_synergy(
    engine: Engine,
    commander_name: str,
    http_client: httpx.Client,
    *,
    ttl_days: int = 7,
    now: datetime | None = None,
) -> list[dict]:
    now = now or datetime.now(timezone.utc)
    slug = slugify(commander_name)
    cached = _cached_row(engine, slug)

    if cached is not None:
        fetched_at = cached["fetched_at"]
        if fetched_at.tzinfo is None:
            fetched_at = fetched_at.replace(tzinfo=timezone.utc)
        if now - fetched_at < timedelta(days=ttl_days):
            return cached["cardlists_json"]

    try:
        cardlists = _fetch(slug, http_client)
    except (httpx.HTTPError, KeyError, ValueError) as exc:
        if cached is not None:
            return cached["cardlists_json"]
        raise RuntimeError(
            f"Could not fetch EDHREC synergy data for {commander_name!r} and no cached copy exists: {exc}"
        ) from exc

    _store(engine, slug, cardlists, now)
    return cardlists
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/ingestion/test_edhrec_client.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ingestion/edhrec/client.py tests/ingestion/test_edhrec_client.py
git commit -m "feat: add cached on-demand EDHREC synergy client"
```

---

## Task 11: `modules/metagame` wrapper

**Files:**
- Create: `mtg-agent/modules/metagame/synergy.py`
- Test: `mtg-agent/tests/modules/test_metagame.py`

**Interfaces:**
- Consumes: `ingestion.edhrec.client.get_commander_synergy` (Task 10).
- Produces: `modules.metagame.synergy.get_synergy_cards(engine, commander_name: str, http_client, *, category_tag: str | None = None) -> list[dict]` — flattens all `cardviews` across matching category panels into `{"name", "synergy", "num_decks", "category"}` rows, sorted by `synergy` descending.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/modules/test_metagame.py
import httpx

from db.schema.models import metadata
from db.session import get_engine
from modules.metagame.synergy import get_synergy_cards

RESPONSE = {
    "container": {
        "json_dict": {
            "cardlists": [
                {
                    "header": "High Synergy Cards",
                    "tag": "highsynergycards",
                    "cardviews": [
                        {"name": "Card A", "synergy": 0.10, "num_decks": 100},
                        {"name": "Card B", "synergy": 0.30, "num_decks": 200},
                    ],
                },
                {
                    "header": "Creatures",
                    "tag": "creatures",
                    "cardviews": [{"name": "Card C", "synergy": 0.20, "num_decks": 150}],
                },
            ]
        }
    }
}


def _client():
    def handler(request: httpx.Request) -> httpx.Response:
        return httpx.Response(200, json=RESPONSE)

    return httpx.Client(transport=httpx.MockTransport(handler))


def test_get_synergy_cards_flattens_and_sorts():
    engine = get_engine(":memory:")
    metadata.create_all(engine)

    results = get_synergy_cards(engine, "Teysa Karlov", _client())

    assert [r["name"] for r in results] == ["Card B", "Card C", "Card A"]
    assert results[0]["category"] == "highsynergycards"


def test_get_synergy_cards_filters_by_category():
    engine = get_engine(":memory:")
    metadata.create_all(engine)

    results = get_synergy_cards(engine, "Teysa Karlov", _client(), category_tag="creatures")

    assert [r["name"] for r in results] == ["Card C"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/modules/test_metagame.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'modules.metagame.synergy'`

- [ ] **Step 3: Write `modules/metagame/synergy.py`**

```python
import httpx
from sqlalchemy.engine import Engine

from ingestion.edhrec.client import get_commander_synergy


def get_synergy_cards(
    engine: Engine,
    commander_name: str,
    http_client: httpx.Client,
    *,
    category_tag: str | None = None,
) -> list[dict]:
    cardlists = get_commander_synergy(engine, commander_name, http_client)

    rows = []
    for panel in cardlists:
        if category_tag is not None and panel["tag"] != category_tag:
            continue
        for view in panel["cardviews"]:
            rows.append(
                {
                    "name": view["name"],
                    "synergy": view["synergy"],
                    "num_decks": view["num_decks"],
                    "category": panel["tag"],
                }
            )

    return sorted(rows, key=lambda r: r["synergy"], reverse=True)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/modules/test_metagame.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add modules/metagame/synergy.py tests/modules/test_metagame.py
git commit -m "feat: add modules/metagame synergy card flattening"
```

---

## Task 12: `apps/mcp_server` — expose the five tools

**Files:**
- Create: `mtg-agent/apps/mcp_server/server.py`
- Test: `mtg-agent/tests/apps/test_mcp_server.py`

**Interfaces:**
- Consumes: `modules.cards.queries.get_card_by_name`, `search_cards` (Task 7); `modules.rules.legality.check_deck_legality` (Task 8); `modules.metagame.synergy.get_synergy_cards` (Task 11); a module-level `get_rulings` query added in this task.
- Produces: `apps.mcp_server.server.mcp` — a `mcp.server.fastmcp.FastMCP` instance with tools `get_card`, `search_cards`, `get_rulings`, `check_deck_legality`, `get_commander_synergy` registered.
- Produces: `apps.mcp_server.server.get_rulings(engine, card_name: str) -> list[dict]`.

- [ ] **Step 1: Write the failing test**

```python
# mtg-agent/tests/apps/test_mcp_server.py
import httpx

from db.schema.models import cards, metadata, rulings
from db.session import get_engine
from apps.mcp_server.server import build_server


def _seeded_engine():
    engine = get_engine(":memory:")
    metadata.create_all(engine)
    with engine.begin() as conn:
        conn.execute(
            cards.insert().values(
                scryfall_id="1", oracle_id="o1", name="Sol Ring", mana_cost="{1}", cmc=1.0,
                colors="", color_identity="", type_line="Artifact", oracle_text="", power=None,
                toughness=None, loyalty=None, set_code="cmr", collector_number="483", rarity="uncommon",
                price_usd=1.85, price_usd_foil=None, price_eur=None, price_tix=None, raw_json={},
            )
        )
        conn.execute(
            rulings.insert().values(
                oracle_id="o1", published_at="2004-10-04", source="wotc", comment="No colored mana."
            )
        )
    return engine


def test_build_server_registers_all_five_tools():
    engine = _seeded_engine()
    http_client = httpx.Client(transport=httpx.MockTransport(lambda r: httpx.Response(200, json={})))

    server = build_server(engine, http_client)

    tool_names = {t.name for t in server._tool_manager.list_tools()}
    assert tool_names == {
        "get_card",
        "search_cards",
        "get_rulings",
        "check_deck_legality",
        "get_commander_synergy",
    }


def test_get_card_tool_returns_seeded_card():
    engine = _seeded_engine()
    http_client = httpx.Client(transport=httpx.MockTransport(lambda r: httpx.Response(200, json={})))
    server = build_server(engine, http_client)

    tool = next(t for t in server._tool_manager.list_tools() if t.name == "get_card")
    result = tool.fn(name="Sol Ring")
    assert result["name"] == "Sol Ring"


def test_get_rulings_tool_returns_seeded_ruling():
    engine = _seeded_engine()
    http_client = httpx.Client(transport=httpx.MockTransport(lambda r: httpx.Response(200, json={})))
    server = build_server(engine, http_client)

    tool = next(t for t in server._tool_manager.list_tools() if t.name == "get_rulings")
    result = tool.fn(card_name="Sol Ring")
    assert len(result) == 1
    assert result[0]["comment"] == "No colored mana."
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/apps/test_mcp_server.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'apps.mcp_server.server'`

- [ ] **Step 3: Write `apps/mcp_server/server.py`**

```python
import httpx
from mcp.server.fastmcp import FastMCP
from sqlalchemy import select
from sqlalchemy.engine import Engine

from db.schema.models import rulings as rulings_table
from modules.cards.queries import get_card_by_name, search_cards
from modules.metagame.synergy import get_synergy_cards
from modules.rules.legality import check_deck_legality


def get_rulings(engine: Engine, card_name: str) -> list[dict]:
    card = get_card_by_name(engine, card_name)
    if card is None:
        return []
    with engine.connect() as conn:
        rows = conn.execute(
            select(rulings_table).where(rulings_table.c.oracle_id == card["oracle_id"])
        ).fetchall()
    return [dict(r._mapping) for r in rows]


def build_server(engine: Engine, http_client: httpx.Client) -> FastMCP:
    server = FastMCP("mtg-agent-cards-rules")

    @server.tool()
    def get_card(name: str) -> dict | None:
        """Look up a single card by exact (case-insensitive) name."""
        return get_card_by_name(engine, name)

    @server.tool()
    def search_cards_tool(
        color_identity_subset: list[str] | None = None,
        type_contains: str | None = None,
        legal_in: str | None = None,
    ) -> list[dict]:
        """Search cards by color identity subset, type text, and/or format legality."""
        return search_cards(
            engine,
            color_identity_subset=color_identity_subset,
            type_contains=type_contains,
            legal_in=legal_in,
        )

    search_cards_tool.__name__ = "search_cards"

    @server.tool()
    def get_rulings_tool(card_name: str) -> list[dict]:
        """Get all known rulings for a card by name."""
        return get_rulings(engine, card_name)

    get_rulings_tool.__name__ = "get_rulings"

    @server.tool()
    def check_deck_legality_tool(
        commander_names: list[str], deck_card_names: list[str], format: str = "commander"
    ) -> dict:
        """Check whether a deck is legal: card count, singleton, color identity, per-format legality."""
        return check_deck_legality(
            engine, commander_names=commander_names, deck_card_names=deck_card_names, format=format
        )

    check_deck_legality_tool.__name__ = "check_deck_legality"

    @server.tool()
    def get_commander_synergy_tool(commander_name: str, category_tag: str | None = None) -> list[dict]:
        """Get EDHREC synergy cards for a commander, optionally filtered to one category."""
        return get_synergy_cards(engine, commander_name, http_client, category_tag=category_tag)

    get_commander_synergy_tool.__name__ = "get_commander_synergy"

    return server


if __name__ == "__main__":
    from db.session import get_engine

    _engine = get_engine()
    with httpx.Client(timeout=15.0) as _http_client:
        build_server(_engine, _http_client).run()
```

Note: `FastMCP.tool()` registers under the *decorated function's* `__name__`. Renaming after decoration (`fn.__name__ = "..."`) is a deliberate workaround so the public tool name matches the spec (`search_cards`, `get_rulings`, etc.) rather than the Python-safe internal name (avoiding a name clash with the imported `search_cards`/`check_deck_legality` functions in the same module). Confirm this rename approach actually renames the registered tool (not just the Python function object) against the installed `mcp` package version — if `FastMCP.tool()` reads `__name__` at decoration time rather than call time, pass `@server.tool(name="search_cards")` instead (check the installed version's signature; both patterns exist across `mcp` SDK versions). If the installed version supports `name=`, prefer it — it's more explicit than the rename workaround.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/apps/test_mcp_server.py -v`
Expected: PASS. If it fails specifically on tool names not matching due to the `__name__` rename not taking effect, switch every `@server.tool()` above to `@server.tool(name="...")` with the intended public name and remove the `__name__` reassignment lines, then re-run.

- [ ] **Step 5: Commit**

```bash
git add apps/mcp_server/server.py tests/apps/test_mcp_server.py
git commit -m "feat: expose cards/rules/metagame as MCP tools"
```

---

## Task 13: Dev subagent definitions

**Files:**
- Create: `mtg-agent/.claude/agents/data-engineer.md`
- Create: `mtg-agent/.claude/agents/rules-engineer.md`
- Create: `mtg-agent/.claude/agents/test-reviewer.md`

**Interfaces:** none (markdown configuration only).

- [ ] **Step 1: Write `.claude/agents/data-engineer.md`**

```markdown
---
name: data-engineer
description: Use for anything touching ingestion/scryfall, ingestion/edhrec, db/schema, or db/migrations in mtg-agent — bulk data ingestion, DB schema changes, migrations, and the EDHREC cache client.
tools: Read, Edit, Write, Bash, Glob, Grep
---

You own the data layer of `mtg-agent`: `ingestion/scryfall/`, `ingestion/edhrec/`, `db/schema/`, `db/migrations/`.

Ground rules:
- Schema changes go through an Alembic migration (`db/migrations/versions/`), never hand-edited into an existing running DB.
- Keep `db/schema/models.py` and the latest migration in sync — a table defined in one and missing from the other is a bug.
- No SQLite-only column types or functions — everything must be portable to Postgres (see the sub-project 1 spec's Data model section).
- `ingestion/edhrec` calls a real, undocumented third-party endpoint. Never widen it into bulk/proactive crawling without going back through a design discussion first — the whole point of the on-demand-cached design was minimizing request volume and risk.
- Every ingestion function gets a test using fixture/mocked HTTP data — no test may make a real network call.

Reference: `docs/superpowers/specs/2026-08-22-cards-rules-edhrec-design.md`.
```

- [ ] **Step 2: Write `.claude/agents/rules-engineer.md`**

```markdown
---
name: rules-engineer
description: Use for anything touching modules/rules or evaluations/legality in mtg-agent — deterministic Commander rules/legality logic and its evaluation suite.
tools: Read, Edit, Write, Bash, Glob, Grep
---

You own `modules/rules/` and `evaluations/legality/` in `mtg-agent`. This is the "judge" logic: deck-legality checks (card count, singleton, color identity, per-format legal/banned status) implemented as real, deterministic code — not LLM inference.

Ground rules:
- Every new rule or edge case gets both a unit test (`tests/modules/test_rules.py`) and, if it's a legality-relevant scenario, a case in `evaluations/legality/cases.py`.
- Cite the actual Commander rule being encoded (in a code comment only if the rule is genuinely non-obvious — most of this logic should be self-explanatory from names).
- If a rules question can't be answered deterministically from the DB (e.g. a genuinely ambiguous multiplayer timing interaction), that's out of scope for this module — it belongs to LLM judge-mode reasoning in a Project/agent context, not hardcoded here.

Reference: `docs/superpowers/specs/2026-08-22-cards-rules-edhrec-design.md`.
```

- [ ] **Step 3: Write `.claude/agents/test-reviewer.md`**

```markdown
---
name: test-reviewer
description: Use to review test coverage across mtg-agent's data-engineer and rules-engineer work — checks for missing edge cases, real network calls in tests, and weak assertions.
tools: Read, Glob, Grep, Bash
---

You review, but do not write, tests in `mtg-agent`. Given a diff or a set of recently changed files, check:

- Every ingestion/client function that makes an HTTP call has a test using `httpx.MockTransport` or equivalent — flag any test that could reach a real network endpoint.
- Every branch in `modules/rules/legality.py` (each violation type) has a corresponding test case, and ideally an `evaluations/legality` case too.
- Assertions check actual values, not just "no exception raised" or "result is not None."
- New DB tables or columns (`db/schema/models.py`) have a corresponding Alembic migration test.

Report gaps as a concrete list (file, missing case, suggested test), not a general "looks fine."
```

- [ ] **Step 4: Verify all three files exist and are valid markdown with frontmatter**

Run: `python -c "import pathlib; [print(p, p.exists()) for p in pathlib.Path('.claude/agents').glob('*.md')]"`
Expected: three `True` lines.

- [ ] **Step 5: Commit**

```bash
git add .claude/agents
git commit -m "chore: scaffold data-engineer, rules-engineer, test-reviewer subagents"
```

---

## Task 14: `CLAUDE.md`

**Files:**
- Create: `mtg-agent/CLAUDE.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Write `CLAUDE.md`**

```markdown
# mtg-agent

A Level 3 Magic: The Gathering Commander assistant: a real database, MCP server, and data ingestion pipelines — built incrementally alongside `../mtg-commander-agent/` (the Level 1 Claude Project package, which stays the live daily driver on phone/web until this repo is proven reliable).

Design spec for the current sub-project: `docs/superpowers/specs/2026-08-22-cards-rules-edhrec-design.md`.

## Layout

- `modules/{cards,rules,metagame}/` — query and business logic, one responsibility per package.
- `ingestion/{scryfall,edhrec}/` — data acquisition. Scryfall is bulk-data, official, deterministic. EDHREC is on-demand, cached, hand-written against an undocumented endpoint — treat it as fragile by design, not a bug if it needs occasional fixing.
- `db/schema/` — SQLAlchemy Core table definitions (source of truth for the shape of the data). `db/migrations/` — Alembic migrations (source of truth for how to get there from any existing DB).
- `apps/mcp_server/` — the MCP server exposing everything above as tools. Directory is named `mcp_server` (underscore) rather than `mcp-server` (the spec's directory name) because it must be a valid Python import path — see Task 1 of the implementation plan for this note.
- `evaluations/legality/` — fixed-case correctness suite for deck-legality logic.
- `tests/` — mirrors the package layout; every ingestion/HTTP function is tested against fixtures, never a live network call.
- `.claude/agents/` — `data-engineer` (ingestion + schema), `rules-engineer` (rules logic + legality evals), `test-reviewer` (coverage review). `deck-engineer` doesn't exist yet — added when the `decks` sub-project starts.

## Stack

Python 3.11+, SQLAlchemy 2.x Core (not ORM), Alembic, httpx, the official `mcp` SDK, pytest.

## Working here

- Local dev only for now: SQLite file at `data/mtg.sqlite` (gitignored), MCP server run via Claude Code/Desktop config. Hosting for mobile/Project access is a deliberately deferred decision — don't build toward a specific host speculatively.
- No live network calls in tests, ever. Fixture/mock everything.
- Say plainly when data is missing, stale, or a fetch failed — never fabricate a plausible-looking answer. This carries over directly from the Level 1 package's philosophy.
- Before adding a new ingestion source or module, check whether it belongs in the current sub-project's spec or is scope creep for a later one — sub-project boundaries exist on purpose (see this repo's specs for the reasoning behind sub-project 1's scope, including EDHREC being folded in after research showed it needed a different-risk-profile treatment than Scryfall).
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add top-level CLAUDE.md for mtg-agent"
```

---

## Final verification (after all tasks)

- [ ] Run the full test suite: `pytest -v` from `mtg-agent/` — all tests across all 14 tasks pass.
- [ ] Run `python -m ingestion.scryfall.run` against the real Scryfall bulk-data endpoints once (this is the one deliberate live-network step in the whole plan, run manually, not as part of any test) and confirm it completes and populates `data/mtg.sqlite`.
- [ ] Run `python -m evaluations.legality.run_eval` against that real DB and confirm all cases still pass against real card data (the fixture-based unit test already covers this against synthetic data — this step is the sanity check against the real dataset).
- [ ] Start the MCP server locally via Claude Code/Desktop config pointing at `apps/mcp_server/server.py`, and manually exercise all five tools from a chat, including one real `get_commander_synergy` call for a commander you actually play.
