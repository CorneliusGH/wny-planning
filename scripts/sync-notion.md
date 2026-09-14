# Sync properties.json from Notion

Procedure for the scheduled "WNY site sync" task. Two Notion databases are the source
of truth; this repo's properties.json (repo root) is a generated rendering of them —
never hand-edited.

| Notion database | data source | JSON group | status |
| --- | --- | --- | --- |
| Buffalo Duplex Table | f772ef05-69dd-49cb-8e8d-112a4bc4874e | `duplex` | PRIMARY |
| WNY Property Table | d010806f-724b-4614-93f5-31626bc72c33 | `resort-site` | secondary (search paused) |

Data flow: Zillow favorites -> the Zillow monitor task -> these two Notion tables ->
properties.json -> GitHub Pages (brief/ and calculator/).

Two files under data/ are NOT produced by this sync and must never be overwritten by it:

- `data/crime-scores.json` — zip-level violent-crime baseline, read by the site pages
  as a labelled fallback when a property has no block-level Crime Score.
- `data/crime-grid.json` — block-level score grid, uploaded by the repo owner and
  consumed by an external monitor process. No site page reads it.

Steps:
1. Query all rows of BOTH Notion data sources via the notion MCP server.
   - WNY Property Table: if the query returns fewer than 40 rows, ABORT without
     committing — a partial read must never overwrite good data.
   - Buffalo Duplex Table: zero rows is valid (the table is new). A failed or errored
     query is not the same as an empty one — on an error, ABORT.
2. Read the current properties.json at the repo root.
3. Rebuild the resort-site entries, one per WNY Property Table row, with this mapping
   (Notion property -> JSON field):
   - id: the digits before "_zpid" in Zillow Link
   - group: "resort-site"
   - address: Property title up to the first ", "
   - town: Town; zip: the 5-digit number in the Property title; county: County
   - price: Price; acreage: Acres; pricePerAcre: Price per Acre
   - bufMiles / bufMinutes: BUF Miles est / BUF Minutes est
   - taxesPerYear: Taxes per Year; landStatus: Land Status
   - water: Water; waterDetails: Water Details
   - zoning: Zoning; zoningNotes: Zoning Notes; utilities: Utilities
   - rank: Rank; overallScore: Overall Score; cashScore: Cash Score
   - speedToRevenue: Speed to Revenue; estCashToEnter: Est Cash to Enter
   - financingPath: Financing Path; daysOnZillow: Days on Zillow; priceCut: Price Cut
   - revenue: {strDwelling, huntingLease, timber, farmHayLease, solarLease,
     mapleSugarbush, gasOilIncome, campingGlamping, eventsBarn, pondFishing} from
     STR Dwelling, Hunting Lease, Timber, Farm Hay Lease, Solar Lease,
     Maple Sugarbush, Gas Oil Income, Camping Glamping, Events Barn, Pond Fishing —
     converting the marks: ✓ -> "yes", ○ -> "maybe", ✗ -> "no", empty -> null
   - zillowLink: Zillow Link
   - photo: the Notion Photo property if set; otherwise preserve the photo value
     from the current properties.json entry with the same id; otherwise null
   - notes: Notes
   - type and sqft: preserve from the current properties.json entry with the same
     id; for a row with no existing entry, type "land" and sqft null unless the
     Notion Notes/Land Status clearly indicate a house
4. Rebuild the duplex entries, one per Buffalo Duplex Table row, with this mapping
   (Notion property -> JSON field, camelCase):
   - id: the digits before "_zpid" in Zillow Link
   - group: "duplex"
   - address: Property title up to the first ", "
   - town and zip: from the Property title
   - neighborhood: Neighborhood; schoolDistrict: School District
   - price: Price; unitsConfig: Units Config; sqft: Sqft; yearBuilt: Year Built
   - occupancy: Occupancy; currentRent: Current Rent
   - garage: Garage; ac: AC; heating: Heating
   - evCharger: EV Charger; generator: Generator — converting the marks:
     ✓ -> "yes", ○ -> "maybe", ✗ -> "no"
   - bath: Bath; systemsNotes: Systems Notes; roof: Roof
   - taxesPerYear: Taxes per Year
   - extraCostsPerYear: Est Extra Costs per Year
   - parkMiles: Park Miles; parkMinutes: Park Minutes
   - estStrMonthly: Est STR Monthly
   - crimeScore: Crime Score — address-level, 1 (safest) to 10 (most dangerous).
     null means the address is outside Buffalo PD coverage (a suburb), NOT that the
     block is safe. The site pages fall back to the zip baseline in
     data/crime-scores.json where one exists, and otherwise show "No city data".
   - latitude: Latitude; longitude: Longitude
   - maintenanceRisk: Maintenance Risk (1 low - 10 high)
   - maintenanceNotes: Maintenance Notes
   - overallScore: Overall Score; rank: Rank; status: Status
   - daysOnZillow: Days on Zillow; priceCut: Price Cut
   - zillowLink: Zillow Link
   - photo: the Notion Photo property if set; otherwise preserve the photo value
     from the current properties.json entry with the same id; otherwise null
   - notes: Notes
5. Preserve every "buffalo-metro" entry from the current properties.json unchanged.
   That group is legacy: nothing writes to it and no page renders it, but it is not
   to be dropped.
6. Top level: { "lastUpdated": today's date YYYY-MM-DD, "source": "Zillow saved
   homes + Buffalo Duplex Table + WNY Property Table (Notion), synced <date>",
   "properties": [...] }.
7. Validate before writing. On ANY failure, ABORT without committing and say what
   failed:
   - ids are unique across ALL groups, not just within one
   - resort-site: at least 40 entries; every entry has price, acreage and rank
   - duplex: may be empty (0 rows is valid); every entry whose Status is "Active"
     has a price
   - ranks are the Notion values in both groups, never recomputed
8. Overwrite properties.json at the repo root (this is the only copy; there is no
   data/properties.json). Commit directly to main with message "sync properties from
   Notion". If pushing to main is not possible, push a branch, open a PR, and say
   clearly that the PR needs merging.
