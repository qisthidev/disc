# DISC Astro Static App

No-database build of the DISC assessment using Astro. All data is bundled as static JSON generated from `db/disc.sql`.

## Commands

- Install: `npm i`
- Generate data: `npm run gen:data`
- Dev: `npm run dev`
- Build: `npm run build`
- Preview: `npm run preview`

## Notes
- Data files are written to `src/data/*.json`.
- Result page computes scoring client-side and shows mapped pattern.
- Share URLs can include `?p={patternId}` to render OG image statically.