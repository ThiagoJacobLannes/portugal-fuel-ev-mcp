# portugal-fuel-ev-mcp

<!-- mcp-name: io.github.ThiagoJacobLannes/portugal-fuel-ev-mcp -->

MCP server that answers "where is the cheapest place to fill up / charge near me in Portugal?", for combustion cars and electric cars, so Claude (or any MCP client) can look it up from official open data.

**Read-only, no keys, no account.** Fuel prices come from DGEG's official price site; EV chargers, live status and prices come from the MOBI.E network feeds on Portugal's National Access Point. Not affiliated with DGEG, MOBI.E or IMT.

## Tools

### `fuel_prices`
Cheapest stations for one fuel type, cheapest first.

Parameters:
- `fuel_type` — a DGEG fuel name or id (`"Gasóleo simples"`, `"Gasolina simples 95"`, `"Gasolina 98"`, `"GPL Auto"`...) or plain words: `"diesel"`, `"petrol"`, `"95"`, `"98"`, `"lpg"`, `"cng"`
- `district` — e.g. `"Lisboa"`, `"Porto"`, `"Faro"` (case- and accent-insensitive)
- `municipality` — e.g. `"Sintra"`, `"Évora"` (overrides `district`)
- `lat` / `lon` + `radius_km` — search around a point (default radius 10 km)
- `brand` — only one brand, e.g. `"Galp"`, `"Prio"`
- `limit` — how many stations (default 10, max 50)

With no area it searches all of Portugal. Each station has: price (EUR per litre, or per kg/m³ for gas), brand, name, address, municipality, coordinates, distance (when searching around a point) and the station's last price update. The result also gives how many stations matched and the min/median/max price.

### `fuel_station_detail`
One station: every fuel it sells with price and last update, opening hours, services and payment methods.

### `fuel_types`
The DGEG fuel types (id, name, unit) and the districts. Only needed when unsure which fuel name to use.

### `ev_chargers`
Public charging sites near a point, nearest first, with live status and published prices.

Parameters:
- `lat` / `lon` — where to search
- `radius_km` — default 5, max 100
- `min_power_kw` — e.g. `50` for fast DC
- `connector` — `"CCS"`, `"Type 2"`, `"CHAdeMO"`, `"CEE"`, or `"DC"` / `"AC"`
- `available_only` — only points that are free right now
- `limit` — sites to return (default 10, max 50)

Each site has: operator, address, distance, opening hours and its matching points. Each point has: live status (available, in use, out of order...), maximum power, connectors and the price the feed publishes (`per_kwh`, `per_minute`, `per_session`, in EUR). The result also says how fresh the data is.

### `ev_charger_detail`
One charging site by site id, point id or EVSE id: all its points with status, power, connectors and price, plus the operator's contact, opening hours and how you can start a charge (app, RFID card, credit card...).

## About EV prices

The prices are the `rates` that the MOBI.E status feed publishes for each charging point: a pricing type (per kWh, per minute of charging, per session, or "other", such as parking time) and an amount. **They may not be the whole bill:**

- The feed does not say which tariff it is (regular, or ad hoc / pay at the charger), nor whether VAT is included. A spot check against MOBI.E's own tariff list found that where a point has an ad hoc tariff the feed usually carries that one; elsewhere it carries the operator's regular tariff.
- On the MOBI.E network the operator's tariff is only the charging point's fee. When you pay with a MOBI.E card or app, your provider (CEME) adds its own energy price, plus network access charges, the MOBI.E management fee and taxes.

Every `ev_chargers` and `ev_charger_detail` result carries this note (`price_note`). Use the prices to compare chargers, and check the operator's app or your provider for the final cost.

## Install

**Quickest** (needs [uv](https://docs.astral.sh/uv/)): no clone needed, it runs straight from PyPI.

```bash
claude mcp add portugal-fuel-ev -- uvx portugal-fuel-ev-mcp
```

For Claude Desktop, use `"command": "uvx", "args": ["portugal-fuel-ev-mcp"]`.

**From source:**

Requires Python 3.10+.

```bash
git clone https://github.com/ThiagoJacobLannes/portugal-fuel-ev-mcp
cd portugal-fuel-ev-mcp
python -m pip install -e .
```

On Windows, if you keep code in a cloud-synced folder (Google Drive, OneDrive), create the virtual environment **outside** it: sync clients drop `desktop.ini` files into folders, which can crash packages on startup.

## Add to Claude Code

```bash
claude mcp add portugal-fuel-ev -- python -m portugal_fuel_ev_mcp.server
```

## Add to Claude Desktop

Merge `claude_desktop_config.snippet.json` into your Claude Desktop `claude_desktop_config.json`, then restart Claude Desktop:

```json
{
  "mcpServers": {
    "portugal-fuel-ev": {
      "command": "python",
      "args": ["-m", "portugal_fuel_ev_mcp.server"]
    }
  }
}
```

## First EV call is slow

The MOBI.E feeds are large (about 180 MB for the inventory, 40 MB for the live status) and are not compressed. The first EV call of the day downloads them, which takes one to three minutes depending on the connection and on MOBI.E's server. The server reads them as they stream in (it never holds the whole file in memory) and keeps a compact cache of about 1 MB, so later calls take milliseconds.

- The inventory is refreshed at most every 24 hours, and the live status every 10 minutes, only when a tool needs them. Each result's `data_freshness` says when the data was published and fetched.
- To download ahead of time (for example if your client times out on long calls), run:

  ```bash
  python -m portugal_fuel_ev_mcp.server --refresh
  ```

- The cache lives in your user cache folder (via `platformdirs`). Set `PORTUGAL_FUEL_EV_CACHE_DIR` to use another folder.
- If a refresh fails, the server keeps using the last cache and says so in `data_freshness.warnings`.

Fuel prices are fetched live from DGEG on each call (cached for 10 minutes).

## Data sources and licences

| Data | Source | Terms |
|---|---|---|
| Fuel prices, stations, opening hours | DGEG, [Preços dos Combustíveis Online](https://precoscombustiveis.dgeg.gov.pt/) (JSON API `.../api/PrecoComb/`) | DGEG's site says the information is free and can be used freely; reuse must name DGEG as the source, show the date and time of the last price update, and reproduce the information faithfully. The tools return `source` and each price's `last_update` for that reason. |
| EV charging sites, points, connectors, live status and prices | MOBI.E, via Portugal's National Access Point ([NAP, IMT](https://nap-portugal.imt-ip.pt/)): `https://pgm.mobie.pt/integration/nap/evChargingInfra` and `.../evActualStatus` (DATEX II 3.3 XML) | Listed on the NAP (datasets 148 and 149) as "Livre acesso" (open access), with no separate licence document. Credit "MOBI.E / NAP Portugal" when you reuse the data. |

Prices are posted by station operators and charging operators; they can be wrong or out of date. Check at the pump or in the charger's app.

The DGEG endpoints are the ones used by the Home Assistant integration [`netsoft-ruidias/ha-custom-component-precoscombustiveis`](https://github.com/netsoft-ruidias/ha-custom-component-precoscombustiveis) and by [`GeiserX/Pumperly`](https://github.com/GeiserX/Pumperly). The DATEX II parsing follows the notes in [`CarregaErvilhas/nap-dashboard`](https://github.com/CarregaErvilhas/nap-dashboard) and [`mobilidade-dados-pt/carregamento-pt`](https://github.com/mobilidade-dados-pt/carregamento-pt).

## How it works

- **Fuel:** `PesquisarPostos` returns every station selling one fuel type in an area, with price (`"1,899 €"`, turned into `1.899`), coordinates and last update; for a point search the server fetches the whole country (about 1 MB) and filters by distance. `GetDadosPosto` gives one station's details.
- **EV:** the two DATEX II feeds are parsed incrementally with Python's built-in `xml.etree.ElementTree.XMLPullParser`, one charging site at a time. Status and prices are joined to the inventory by (site id, point id), because some operators reuse bare point ids across sites. Requests send `If-None-Match`, so an unchanged feed is not downloaded again.

If a source changes or is down, tools return `{"error": ...}` instead of wrong data.

## License

MIT
