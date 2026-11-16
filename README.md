# Earth analogs for permanent Moon base locations and Mars

NASA Space Apps Challenge 2026 — team J4Ks (Tegucigalpa, Honduras).

We look for places on Earth that are analogs of the candidate permanent Moon base regions and of Mars, comparing
several physical variables at once (slope, terrain roughness, temperature, aridity, permafrost, wind, dust,
mineralogy) and stating, for every analog, **what matches and what differs** (gravity, atmosphere, pressure and scale
included).

## What is in here

- Data sources: [#data-sources-and-licenses](#data-sources-and-licenses)
- Third-party licenses: [#third-party-software-and-licenses](#third-party-software-and-licenses)
- AI tools used: [#ai-tools-used](#ai-tools-used)
- Landing page: [docs/index.html](docs/index.html)

## Architecture

| Part | Where | What it does |
| :-- | :-- | :-- |
| ETL | [etl/](etl/) | Builds one table of Earth land cells (HEALPix) with one column per variable, offline. |
| Search engine | [motor/](motor/README.md) | Rust. Compares every cell with each Moon and Mars target and writes `metrics.json`. |
| Viewer | [visor/](visor/README.md) | CesiumJS globe with candidates, pins and a sheet of what matches and what differs. |
| Agents and AI | [scripts/mcp/](scripts/mcp/) | MCP server for AI agents, local bridge, and a bring-your-own-key assistant in the viewer. |

## WebAssembly: the engine scoring inside the browser

`motor/wasm` wraps the same `nucleo` profile in an `extern "C"` ABI (no `wasm-bindgen`, so the artifact is one `.wasm`
of ~36 KB that loads with `WebAssembly.instantiateStreaming`). JavaScript writes the profile and the cells into Wasm
linear memory, gets back `S` and `D²` per cell, and frees explicitly.

```bash
cd motor && cargo build -p wasm --target wasm32-unknown-unknown --release
node visor/pruebas/wasm/paridad.mjs    # nativo vs wasm, 1e-9
```

The parity test scores the four cells of `motor/wasm/src/datos_prueba.rs` twice — once with the native binary, once
with the `.wasm` — and fails if `S` or `D²` differ. That is the guarantee: one formula, two targets.

What stays native is the sweep: 58.7 million cells × 6 variables in `f32` are 1.41 GB, and `memmap2` on
`wasm32-unknown-unknown` is a stub that returns `Unsupported` while `rayon` falls back to a single thread. So the
browser re-scores candidates that the native engine already prefiltered, instead of scanning the planet.

## Run everything with one command

```bash
node scripts/iniciar.mjs --run /path/to/run --puntos /path/to/other-run
```

Starts the local bridge (engine and assistant), starts the viewer on the first free port and opens the page. `RUN`
and `RUN_PUNTOS` can come from the environment instead of the flags. Without `RUN` the viewer still works; only the
"recalculate with the Rust engine" button is off. Options: `--build` serves the production build, `--sin-abrir`
does not open the browser, `--sin-puente` skips the bridge. Ctrl+C stops everything.

## Use it from an AI agent (MCP)

```bash
node scripts/mcp/servidor-mcp.mjs   # stdio MCP server, no npm dependencies
```

`.mcp.json` registers it for Claude Code. Setup for Claude Desktop, Codex CLI, Cursor and Antigravity, the tool list
and example prompts: [#agent-mode-mcp-and-the-ai-assistant](#agent-mode-mcp-and-the-ai-assistant).

## Ask about a candidate with your own AI key

The viewer's **Asistente IA** panel works without a key (deterministic summary) or with your own Gemini, OpenAI,
DeepSeek or Claude key. The key stays in the page's memory only: never on disk, in `localStorage` or in the URL. See
[#key-handling](#key-handling-for-shared-machines-and-judging-vms).

## Data sources and licenses

Open datasets the pipeline reads. Raw files are kept outside the repository under `$DATOS` and never enter git.
Each entry lists product and version, where it comes from and its license.

### Earth

| Data | Product and version | Source | License |
| :-- | :-- | :-- | :-- |
| Elevation and water mask | Copernicus DEM GLO-90, release 2024_1 | AWS Open Data `copernicus-dem-90m` | Copernicus DEM license |
| Emissivity / silica | ASTER GED AG1km V003 | NASA Earthdata (LP DAAC) | NASA open data |
| Land surface temperature | MODIS Aqua MYD11C3 V061, monthly, 2013–2022 | NASA Earthdata (LP DAAC) | NASA open data |
| Vegetation | MODIS MOD13C1 V061 (NDVI), 2013–2022 | NASA Earthdata (LP DAAC) | NASA open data |
| Wind (friction velocity, air density) | MERRA-2 `M2T1NXFLX` | NASA GES DISC | NASA open data |
| Dust emission | MERRA-2 `M2TMNXADG` (`DUEM001`–`DUEM005`) | NASA GES DISC | NASA open data |
| Dust storm frequency | `MODFDS_MON_GLB_L3` v1 | NASA GES DISC | NASA open data |
| Aridity | Global Aridity Index and PET v3.1 | Figshare 7504448 | CC BY 4.0 (non-commercial use stated) |
| Permafrost | Obu et al. 2018, mean annual ground temperature | ESA CCI / PANGAEA | CC BY 3.0 |
| Lithology | GLiM v1.0, 0.5° grid | Hartmann & Moosdorf 2012 | CC BY 3.0 |
| Minerals | EMIT `EMITL3ASA` V002 (0.5°) | NASA Earthdata | NASA open data |
| Surface water | JRC Global Surface Water v1.5 | global-surface-water.appspot.com | Copernicus open license |
| Wetlands and lakes | GLWD v2 | Figshare | CC BY 4.0 |
| Dunes and sand | GSDP30 and GSDS30 | Zenodo 10.5281/zenodo.13907012, 10.5281/zenodo.10441041 | CC BY 4.0 |
| Lava tubes (catalog) | Pyroduct Digital Catalog v3.0 (indices & survey metadata, no WGS84 coordinates) | Zenodo 10.5281/zenodo.14535885 | CC BY 4.0 |
| Lava tube coordinates (map entrances) | OpenStreetMap `natural=cave_entrance` nodes/ways | openstreetmap.org/copyright | ODbL 1.0 |
| Lava tube coordinates (open gazetteer/fauna) | Wikidata P625 & Plazi (`10.15468/x49sxc`) | wikidata.org / Plazi | CC0 1.0 |
| Lava tube coordinates (published caves) | García et al. 2021 (`10.3897/subtbiol.38.61733`); Martínez et al. 2026 (`10.15468/eugwe2`); Nicolosi et al. (`10.15468/f4xjdb`); Bell et al. 2022 DRUM (`10.13016/ovqx-j09w`) | Pensoft / GBIF / UMD DRUM | CC BY 4.0 |
| Ground ice | Karjalainen et al. 2022 v1.1 | Zenodo 10.5281/zenodo.7009875 | CC BY 4.0 |
| Sky-view factor | Li et al. 2024 | ESSD | CC BY 4.0 |
| Soil fine fraction | SoilGrids 2.0 (clay, silt), 1 km | ISRIC | CC BY 4.0 |

### Moon (south pole)

| Data | Product and version | Source | License |
| :-- | :-- | :-- | :-- |
| Elevation | LOLA `LDEM_80S_20M` V2.1 | PDS Geosciences Node | NASA open data |
| Temperature | Diviner GDR L3 polar, `dgdr_tbol_avg_pols`; Paige et al. 2010 PRP | PDS Geosciences Node | NASA open data |
| Thermal inertia (context) | Diviner H-parameter, Hayne et al. 2017 | PDS Geosciences Node | NASA open data |
| Silica | Diviner Christiansen Feature, Lucey et al. 2021 | Zenodo 10.5281/zenodo.4558194 | CC BY 4.0 |
| Local slopes at 5 m | LOLA site DEMs, Barker et al. 2021 | NASA PGDA | NASA open data |
| Illumination | LOLA polar illumination and permanent shadow, Mazarico et al. 2011 | NASA PGDA | NASA open data |
| Radar roughness (context) | LRO Mini-RF CDR mosaic V1.0 | PDS Geosciences Node | NASA open data |
| Artemis candidate regions | USGS, DOI 10.5066/P1MEQ6UK | USGS | Public domain |

### Mars

| Data | Product and version | Source | License |
| :-- | :-- | :-- | :-- |
| Elevation | HRSC-MOLA blended DEM v2 | USGS Astrogeology | Public domain |
| Local elevation | CTX and HiRISE DTMs | USGS STAC | Public domain |
| Minerals | TES global mineral maps, Bandfield 2002 | tes.asu.edu | NASA open data |
| Thermal inertia | THEMIS global thermal inertia; TES, Putzig & Mellon 2007 | USGS Astrogeology / PDS | Public domain |
| Dust (MDAD) | Mars Dust Activity Database v1.1 | Zenodo 10.5281/zenodo.7480334 | CC BY 4.0 |
| Atmospheric dust | Montabone dust climatology v2 (MY24–MY36) | LMD | CC BY-SA 3.0 |
| Climate (local use only) | Mars Climate Database 6.2 | LMD | Scientific use; not redistributed |
| Caves | MGC³, Cushing 2015, doi:10.17189/1519222 | NASA PDS | Public domain |

## Third-party software and licenses

Code in this repository is [MIT](LICENSE); this documentation is CC BY 4.0. Every third-party component and its license:

Inventory of the third-party code this project compiles, runs or ships. The pinned, complete list lives in the lock
files (`motor/Cargo.lock`, `visor/package-lock.json`); this page records the licenses that apply and how to reproduce
the full inventory.

Everything below is open source and OSI-approved (MIT, Apache-2.0, BSD, Zlib, ISC, MPL-2.0). Nothing with a
no-derivatives or non-commercial clause is part of the code that this repository distributes.

### Rust — engine

`motor/nucleo` has **no production dependencies**. `motor/pipeline` and the workspace compile **134 third-party
crates** in total.

| Crate | Version constraint | License | Used for |
| :-- | :-- | :-- | :-- |
| anyhow | 1 | MIT OR Apache-2.0 | Error handling in the binary |
| bytemuck | 1 | Zlib OR Apache-2.0 OR MIT | Reading `f32`/`u32` columns as typed slices |
| cdshealpix | 0.10.1 | Apache-2.0 OR MIT | HEALPix cell centres and vertices |
| memmap2 | 0.9 | MIT OR Apache-2.0 | Memory-mapping the master table |
| rayon | 1 | MIT OR Apache-2.0 | Parallel sweep |
| serde / serde_json | 1 | MIT OR Apache-2.0 | Reading profiles, writing `metrics.json` |
| nalgebra (dev only) | 0.35 | Apache-2.0 | Cross-checking the linear algebra in tests |
| proptest (dev only) | 1 | MIT OR Apache-2.0 | Property tests |

Full inventory with version and license of every crate:

```bash
cd motor && cargo metadata --format-version 1 \
  | python -c "import json,sys; [print(p['name'], p['version'], p.get('license')) for p in json.load(sys.stdin)['packages'] if p.get('source')]"
```

### JavaScript / TypeScript — viewer

**121 packages** are installed by `npm ci`; the four direct ones are:

| Package | Version | License | Used for |
| :-- | :-- | :-- | :-- |
| cesium | 1.147.0 | Apache-2.0 | 3D globe, imagery, entities, primitives |
| vite | ^5.4.10 | MIT | Dev server and production build |
| typescript | ~5.6.2 | Apache-2.0 | Type checking |
| vite-plugin-static-copy | ^2.1.0 | MIT | Copies Cesium's `Assets`, `Workers`, `Widgets`, `ThirdParty` into the build |

Reproduce with `cd visor && npm ci && npx license-checker --summary`. CesiumJS ships its own `ThirdParty` notices with
the build; they are redistributed unmodified.

### Python — ETL (not shipped, run offline)

| Package | License | Used for |
| :-- | :-- | :-- |
| numpy | BSD-3-Clause | Raster arithmetic and statistics |
| scipy | BSD-3-Clause | Filters and interpolation |
| astropy | BSD-3-Clause | FITS/WCS handling for planetary products |
| cdshealpix | Apache-2.0 | HEALPix indexing |
| GDAL (osgeo) | MIT/X11 | Reading GeoTIFF, PDS, NetCDF and HDF |

### Imagery shown in the globe (not redistributed)

| Service | License | Note |
| :-- | :-- | :-- |
| Sentinel-2 cloudless (EOX, s2maps.eu) | CC BY-NC-SA 4.0 | Optional background, ~10 m. Non-commercial: the project is a non-profit hackathon and the tiles are requested at runtime, never redistributed |
| NASA GIBS Blue Marble Next Generation | NASA open data | Default public-domain fallback, no restrictions |

### Datasets

Every dataset keeps its own license; the complete table (product, version, source and license) is in
[data-sources.md](data-sources.md). Raw dataset files are not redistributed in this repository.

## AI tools used

The 2026 NASA Space Apps Participant Terms and Conditions require participants to identify the AI tools and other
resources used to build the solution. This is that declaration.

### Tools and scope

| Tool | Team member | Scope of its work | Sessions recorded |
| :-- | :-- | :-- | --: |
| Claude Code (Anthropic) | Fernando (@RedFer855) | Rust engine (`motor/nucleo`, `motor/pipeline`), ETL scripts (`etl/`), MCP server and local bridge (`scripts/mcp/`, `scripts/`), documentation and ADRs | 15 |
| Antigravity (Google, Gemini models) | Guillermo (@IngGuillermoRosa) | Viewer interface (`visor/src/ui`, `visor/src/motor`), panels, layers, sheet layout | 11 |

Each session was recorded with the tool that was used, so the count above is traceable.

### What the humans did

- Chose the challenge, the variable vector, the similarity formula (Mahalanobis with a tolerance zone, Gower mix,
  `S = exp(−γ·D²)`), the output contract and every architecture decision; all of them are written down in the documentation.
- Reviewed every change through pull requests with CI (tests, clippy, gitleaks, repository checks) before it reached
  `main`.
- Verified the arithmetic against an independent implementation in NumPy (`analisis/referencia_motor.py`), row by row,
  on the full table.
- Ran and checked every pipeline execution against the reference implementation.

### What the AI did not do

- **No number in this repository comes from a model.** Every value in the tables, the sheets and `metrics.json` is
  computed from the datasets listed in [data-sources.md](data-sources.md).
- No AI service is called at build time or by the pipeline. The viewer's optional assistant (BYOK) is the only place
  where an external model is invoked, it is off by default, and it needs the user's own key.
- No AI tool had access to credentials. No API key is stored in the repository or in its history.

### Assistant in the viewer (optional, separate from the above)

The **Asistente IA** panel works without a key (deterministic summary generated from the computed facts) or with the
user's own Gemini, OpenAI, DeepSeek or Claude key. The key lives in the page's memory only: never on disk, in
`localStorage` or in the URL. Details: [mcp-and-ai.md](mcp-and-ai.md).

## Agent mode (MCP) and the AI assistant

Two ways to drive the search engine and read its results without opening the code:

1. **Agent mode (MCP):** any MCP client (Claude Code, Claude Desktop, Codex CLI, Cursor, Antigravity) talks to
   `node scripts/mcp/servidor-mcp.mjs` over stdio. It reads the published results and can run the Rust engine with
   all of its options.
2. **Assistant in the viewer:** the web viewer has an "Asistente IA" panel. Without a key it gives a deterministic
   summary built from the engine numbers. With your own Gemini, OpenAI, DeepSeek or Claude key it answers questions
   about the selected candidate using only those numbers.

Both need only Node 22 or newer. There are no npm dependencies.

### Data the tools read

| Variable | What it is | Default |
| :-- | :-- | :-- |
| `EA_DATOS` | Published folder (`metrics.json`, `resultados/`, `puntos.geojson`, `capas7/`) | `$RUN/salida` if it has `metrics.json`, else `visor/public/datos` |
| `RUN` | ETL run folder (`manifest.json`, `columns/`, `perfiles.json`). Needed only to run the engine | unset |
| `RUN_PUNTOS` | Second run with distance columns, passed as `--puntos` when `usar_puntos` is true | unset |
| `EA_SALIDAS` | Where each engine run writes its own folder | `salidas/mcp/` (ignored by git) |
| `EA_PIPELINE` | Engine binary | `motor/target/release/pipeline(.exe)`, then `debug` |

Build the engine once with `cd motor && cargo build --release`. Clients never pass file paths: every path comes from
these variables or from the repository.

### Agent mode (MCP)

#### Tools

| Tool | What it does | Writes? |
| :-- | :-- | :-- |
| `listar_objetivos` | Moon and Mars target sites, profiles and best published score | no |
| `listar_resultados` | Published results (site × profile) and the MCP runs of this session | no |
| `obtener_candidatos` | Earth candidates for a target, filtered by score, bounding box or full coverage | no |
| `explicar_candidato` | Physical sheet of one candidate: what matches, what differs, non-scored diagnostics, nearby lava tubes, craters and salt flats, warnings, and gravity, pressure, atmosphere and radiation | no |
| `consultar_pines` | Lava tubes, impact craters and salt flats by type, box or distance to a point | no |
| `resumir_capas7` | Earth variables at ~50 km (HEALPix level 7) at a point or summarized over a box | no |
| `ejecutar_motor` | Runs `pipeline` with `objetivo`, `perfil`, `candidatos`, `hilos`, `bbox`, `region_geojson`, `perfil_personalizado` (weights and tolerances), `separacion_km`, `excluir_km`, `k_area`, `min_area_km2`, `solo_barrido`, `usar_puntos`, `publicar_visor` | its own run folder; `publicar_visor: true` replaces that target in `visor/public/datos` |
| `ejecutar_escenarios` | Runs `--escenarios` and returns how stable each candidate is across scenarios | its own run folder |
| `estado_corrida` | Status, duration, engine report and top 5 candidates of a run | no |

Only one engine run at a time. `ejecutar_motor` waits up to `esperar_s` (default 50 s) and then returns the run id;
call `estado_corrida` with it. To read a finished run, pass `corrida: "<id>"` to `obtener_candidatos` or
`explicar_candidato`.

Prompts: `evaluar_analogo`, `comparar_objetivos` and `buscar_en_region`. They tell the model to use only tool numbers
and to state what matches and what differs.

#### Claude Code

The repository has `.mcp.json`; open the repo and approve the `earth-analogs` server when asked. Or add it by hand:

```bash
claude mcp add earth-analogs -- node scripts/mcp/servidor-mcp.mjs
```

Set `RUN` in the shell before starting `claude` to let it run the engine.

#### Claude Desktop

Edit `claude_desktop_config.json` (Settings → Developer → Edit Config) and restart the app. Use absolute paths:

```json
{
  "mcpServers": {
    "earth-analogs": {
      "command": "node",
      "args": ["C:/path/to/earth-analogs/scripts/mcp/servidor-mcp.mjs"],
      "env": { "RUN": "C:/path/to/run" }
    }
  }
}
```

#### OpenAI Codex CLI

```bash
codex mcp add earth-analogs --env RUN=/path/to/run -- node /path/to/earth-analogs/scripts/mcp/servidor-mcp.mjs
```

Or in `~/.codex/config.toml`:

```toml
[mcp_servers.earth-analogs]
command = "node"
args = ["/path/to/earth-analogs/scripts/mcp/servidor-mcp.mjs"]
env = { RUN = "/path/to/run" }
```

#### Cursor

Cursor reads `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global), not `.mcp.json`. Copy the same block,
with an absolute path if you use the global file.

#### Antigravity

Open the MCP servers panel → *Manage MCP Servers* → *View raw config* and add the same `earth-analogs` block as
for Claude Desktop, with absolute paths. (Menu names not verified on every version.)

#### Example prompts

- "List the Mars targets and explain the best Earth analog for Jezero (JZ)."
- "Which lava tubes are within 100 km of candidate 3 for CB? What differs physically from the Moon?"
- "Run the engine for JZ only inside 15,-15,35,30 with slope weight 2, then explain the top result."

### Assistant in the viewer (bring your own key)

Open the viewer, pick a candidate and click **Asistente IA**.

- **Sin clave (no key):** deterministic summary of the same numbers the sheet shows. Always available.
- **With a key:** choose the provider, paste the key, click *Usar clave*. The model gets a short JSON with the
  candidate values, the target values, tolerances, labels, nearby features and planetary differences, and an
  instruction not to invent figures.

| Provider | Endpoint | Key travels in | From the browser |
| :-- | :-- | :-- | :-- |
| Google Gemini | `generativelanguage.googleapis.com/v1beta/models/<model>:generateContent` | `x-goog-api-key` header | yes |
| OpenAI | `api.openai.com/v1/chat/completions` | `Authorization: Bearer` | yes |
| DeepSeek | `api.deepseek.com/v1/chat/completions` | `Authorization: Bearer` | only through the local bridge |
| Claude (Anthropic) | `api.anthropic.com/v1/messages` | `x-api-key` | yes (`anthropic-dangerous-direct-browser-access`) |

The model name is editable; defaults change over time.

#### Key handling (for shared machines and judging VMs)

- The key lives only in this page's memory. It is never written to `.env`, disk, `localStorage`, cookies, the URL
  or the console. The input field is `type="password"` and is cleared after loading.
- *Recordar solo en esta pestaña* keeps it in `sessionStorage`, which the browser deletes when the tab closes.
- *Borrar clave* drops it from memory and from `sessionStorage`. Reloading the page also drops it.
- Gemini keys go in the `x-goog-api-key` header, never in `?key=`.
- Every error shown in the panel is passed through a redactor that masks `AIza…`, `sk-…`, `Bearer …` and the key
  itself.
- Model answers are rendered as plain text (`textContent`), never as HTML.
- The team's own keys are never used: without a key the assistant stays deterministic.

### Local bridge (optional)

```bash
RUN=/path/to/run node scripts/mcp/puente-http.mjs
```

It listens on `http://127.0.0.1:5174` only. When the viewer finds it (only when you open the assistant), it:

- lets DeepSeek (or any provider) be called through it, avoiding browser CORS limits;
- shows **Recalcular con motor Rust**, which runs the engine for the selected target, optionally only inside the
  current view, and draws the new candidates without publishing them.

Protections: the `Host` header must be `127.0.0.1` or `localhost` (DNS rebinding), `Origin` must be a localhost
origin or listed in `EA_PUENTE_ORIGENES`, and every `POST` needs `X-Earth-Analogs: 1`. The key arrives in
`X-IA-Clave` on each request, is forwarded to the provider and is not stored or logged; there is no key vault and no
endpoint that returns a key. The bridge never publishes to the viewer.

### Tests

```bash
node --test "scripts/mcp/*.test.mjs"
npm --prefix visor run prueba
```

## NASA Space Apps requirements

Checked against the *2026 NASA Space Apps Participant Terms and Conditions* (https://www.spaceappschallenge.org/legal/):

| Requirement | Where |
| :-- | :-- |
| Original content freely available or OSI-licensed (Apache-2.0 is NASA's default when none is stated) | MIT, root [`LICENSE`](LICENSE) |
| Licenses of all third-party materials identified in the documentation | [#third-party-software-and-licenses](#third-party-software-and-licenses) |
| Every source cited, even open-source and freely available ones | [#data-sources-and-licenses](#data-sources-and-licenses) |
| AI tools and other resources identified | [#ai-tools-used](#ai-tools-used) |
| Publicly accessible, no registration, no password | Public repository; the viewer runs locally with one command and needs no account or key |
| No secrets in the repository | `.env` ignored, gitleaks in CI, no key in history |

