# Sync properties.json from Notion

Procedure for the scheduled "WNY site sync" task. The Notion database "WNY Property
Table" (data source d010806f-724b-4614-93f5-31626bc72c33) is the source of truth for
all resort-site property data. This repo's properties.json (repo root) is a generated
rendering of it — never hand-edited.

Steps:
1. Query all rows of the Notion data source via the notion MCP server. If the query
   returns fewer than 40 rows, ABORT without committing — a partial read must never
   overwrite good data.
2. Read the current properties.json at the repo root.
3. Rebuild the resort-site entries, one per Notion row, with this mapping
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
4. Preserve every "buffalo-metro" entry from the current properties.json unchanged.
5. Top level: { "lastUpdated": today's date YYYY-MM-DD, "source": "Zillow saved
   homes + WNY Property Table (Notion), synced <date>", "properties": [...] }.
6. Validate before writing: unique ids; every resort-site entry has price, acreage,
   and rank; ranks are the Notion values, never recomputed. On any validation
   failure, ABORT without committing and say what failed.
7. Overwrite properties.json at the repo root (this is the only copy; there is no
   data/ folder). Commit directly to main with message "sync properties from
   Notion". If pushing to main is not possible, push a branch, open a PR, and say
   clearly that the PR needs merging.
