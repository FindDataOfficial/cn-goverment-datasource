# fd-cn-gov

[![PyPI version](https://img.shields.io/pypi/v/fd-cn-gov.svg)](https://pypi.org/project/fd-cn-gov/)
[![Python versions](https://img.shields.io/pypi/pyversions/fd-cn-gov.svg)](https://pypi.org/project/fd-cn-gov/)
[![License: MIT](https://img.shields.io/pypi/l/fd-cn-gov.svg)](https://github.com/FindDataOfficial/cn-goverment-datasource/blob/main/LICENSE)

Scrapers + a self-contained datasource registry for **Chinese central-government ministry open-information archives**. Catalog-crawls the public notice / news / data archives of 11 ministries (MOF, PBC, NDRC, MOFCOM, MOHURD, MOT, MOA, SAFE, MNR, MEE, MEM), emitting one JSON record per listed document, and ships a SQLite + JSON registry describing every datasource and its column schema.

Standalone: no monorepo, no MCP server, no `daas.db` dependency. The `MANIFEST` at the top of each scraper is the single source of truth; the bundled `registry.db` / `registry.json` are derived artifacts.

## Ministries

| Name | Label | Seed URL |
|------|-------|----------|
| `mee_gsgg_archive` | MEE Notice Archive (生态环境部公示公告) | https://www.mee.gov.cn/ywdt/gsgg/ |
| `mem_tzgg_archive` | MEM Notice Archive (应急管理部通知公告) | https://www.mem.gov.cn/gk/tzgg/ |
| `mnr_tzgg_archive` | MNR Notice Archive (自然资源部通知公告) | https://www.mnr.gov.cn/gk/tzgg/ |
| `moa_govpublic_archive` | MOA GovPublic Archive (农业农村部 机构分类) | https://www.moa.gov.cn/govpublic/ |
| `mof_gkml_archive` | MOF gkml Archive (财政部信息公开) | https://www.mof.gov.cn/gkml/ |
| `mofcom_xwfb_archive` | MOFCOM News Preview (商务部新闻发布) | https://www.mofcom.gov.cn/xwfb/index.html |
| `mohurd_xinwen_archive` | MOHURD Xinwen Archive (住建部新闻动态) | https://www.mohurd.gov.cn/xinwen/ |
| `mot_shuju_archive` | MOT Data Hub Archive (交通运输部数据) | https://www.mot.gov.cn/shuju/index.html |
| `ndrc_tzgg_archive` | NDRC Notice Archive (发改委通知公告) | https://www.ndrc.gov.cn/xwdt/tzgg/ |
| `pbc_xinwen_archive` | PBC News Archive (人民银行新闻发布) | https://www.pbc.gov.cn/goutongjiaoliu/113456/113469/index.html |
| `safe_whxw_archive` | SAFE News Archive (外汇局外汇新闻) | https://www.safe.gov.cn/safe/whxw/index.html |

Each scraper emits a JSON array of records to stdout with at least `title`, `date` (`YYYY-MM-DD`, from the URL `t<YYYYMMDD>_` token with a `<span>` fallback), and `url` (absolute, the primary key). Some add `section` / `subsection` / `doc_type` / `department`. See `fd-cn-gov describe <name>` for the exact columns of any source.

## Install

```bash
pip install fd-cn-gov
```

Requires Python ≥3.10. Dependencies: `scrapling` (HTTP + adaptive parsing), `sqlalchemy`.

### From source

```bash
pip install git+https://github.com/FindDataOfficial/cn-goverment-datasource.git
```

## CLI

```bash
# List the 11 registered sources
fd-cn-gov list

# Show one source's identity + full column schema
fd-cn-gov describe mof_gkml_archive

# Crawl one archive (default: 50 pages per sub-archive; prints JSON to stdout)
fd-cn-gov crawl mof_gkml_archive --max-pages 2 > records.json

# Full crawl, no page cap (use sparingly — see Polite crawling below)
fd-cn-gov crawl mof_gkml_archive --all > records.json

# Regenerate the bundled registry from the scripts' MANIFESTs
fd-cn-gov build-registry
```

## Python API

```python
import fd_cn_gov

# Discover datasources
for s in fd_cn_gov.list_sources():
    print(s.name, s.label, s.url)

# One source + its column schema
src = fd_cn_gov.get_source("mof_gkml_archive")
cols = fd_cn_gov.get_columns("mof_gkml_archive")  # -> list[Column]
#   Column(name='url', type='string', primary_key=True, nullable=False,
#          description='absolute document URL (.htm/.html/.pdf)',
#          source_field='a@href', semantic_type='url', ...)
```

## Registry schema

The bundled `fd_cn_gov/registry/registry.db` is a 3-table SQLite, mirroring the column shapes of the originating `daas.db` for these sources (no foreign keys, no stale-FK footgun):

- **`sources`** — one row per ministry: `id, name, label, url, description, category, category_label, config_json`
- **`datasource_columns`** — one row per output column: `datasource_id, table_name, column_name, column_type, is_primary_key, is_nullable, description, source_field, unit, semantic_type`
- **`scraw_configs`** — one row per scraper: `name, url, columns_json`

`registry.json` is a deterministic full dump of the same three tables (`sort_keys`), so the registry is diff-friendly in code review.

To regenerate after editing a `MANIFEST`:

```bash
fd-cn-gov build-registry
```

The build is **logically idempotent**: re-running produces an identical `.dump` and a byte-identical `registry.json`. (Only SQLite's file-change-counter header byte differs on each write — unavoidable on any SQLite write; compare via `.dump` or `registry.json` for equality.)

## Polite crawling

These are `.gov.cn` hosts. Each scraper paces requests (`SLEEP ≈ 0.3s` between pages) and defaults to a **50-page cap per sub-archive**. Use `--max-pages N` to bound a crawl further; reserve `--all` for when you genuinely need full history and can afford the time. Do not run multiple `--all` crawls in parallel against the same ministry.

## Project layout

```
fd-cn-gov/
├── pyproject.toml
├── README.md
├── LICENSE
└── fd_cn_gov/
    ├── __init__.py            # public read API: list_sources / get_source / get_columns
    ├── cli.py                 # fd-cn-gov CLI
    ├── scraw_contract.py      # ScrawManifest / ScrawColumn dataclasses (vendored, trimmed)
    ├── build_registry.py      # regenerates registry.db + registry.json from MANIFESTs
    ├── scripts/               # 11 ministry scrapers, each with a module-level MANIFEST
    │   ├── mof_gkml_archive.py
    │   └── ...
    └── registry/
        ├── __init__.py        # read API over the bundled DB
        ├── registry.db        # generated — checked in
        └── registry.json      # generated — checked in
```

## License

MIT
