### DISC Personality Test — Project Documentation and Modern Rebuild Plan

This repository contains a simple web app for administering a DISC personality assessment and mapping the result to a pre-defined pattern description. The legacy stack is PHP + MySQL with no framework. Below is a complete overview of how it works today and a step-by-step plan to rebuild it with a modern stack, keeping the logic identical.

### What this app does
- Renders 28 groups of 4 terms.
- The user selects one “Most” and one “Least” from each group.
- Tallies D/I/S/C counts for “Most” and “Least”, computes the “Change” profile (Most − Least).
- Maps the computed D/I/S/C “Change” values to segments, then to a pattern, then displays the pattern description.

### Key files
- `index.php`: renders the test, reads terms from DB, posts selections
- `result.php`: calculates results and renders the output
- `db/disc.sql`: schema + seed data (dummy terms and patterns)
- `screenshot/`: example UI images

### How it works (flow)
```html
<!-- index.php excerpt -->
<form method='post' action='result.php'>
  <!-- 28 rows × 4 columns of terms -->
  <input type='radio' name='m[<index>]' value='D|I|S|C|#' required />
  <input type='radio' name='l[<index>]' value='D|I|S|C|#' required />
</form>
```

```php
// result.php excerpt
$most  = array_count_values($_POST['m']);
$least = array_count_values($_POST['l']);
$aspect = ['D','I','S','C','#'];
foreach ($aspect as $a) {
  $result[$a]['most']  = isset($most[$a])  ? $most[$a]  : 0;
  $result[$a]['least'] = isset($least[$a]) ? $least[$a] : 0;
  $result[$a]['change'] = ($a!='#' ? $result[$a]['most'] - $result[$a]['least'] : 0);
}
// Lookup segments for Graph III and map to pattern via pattern_map
```

### Database model (legacy)
- `personalities`: term bank, 28 groups of 4 terms
- `results`: maps each dimension’s “Change” value to a segment for a given graph (Graph III is used)
- `pattern_map`: maps four segment values (d,i,s,c) to a `pattern` id
- `patterns`: pattern descriptions that are shown to the user

```sql
CREATE TABLE personalities (
  id TINYINT(2) PRIMARY KEY AUTO_INCREMENT,
  no TINYINT(2) NOT NULL,        -- group number 1..28
  term VARCHAR(100) NOT NULL,     -- placeholder in repo (real terms not included)
  most CHAR(1) NOT NULL,          -- D/I/S/C/# marker for "Most"
  least CHAR(1) NOT NULL          -- D/I/S/C/# marker for "Least"
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

CREATE TABLE results (
  id INT PRIMARY KEY AUTO_INCREMENT,
  dimension CHAR(1) NOT NULL,   -- D/I/S/C
  intensity INT NOT NULL,
  value INT NOT NULL,           -- delta (Most-Least)
  segment INT NOT NULL,         -- bucketed segment
  graph TINYINT(1) NOT NULL     -- graph 1/2/3 (uses 3)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

CREATE TABLE pattern_map (
  d TINYINT(1) NOT NULL,
  i TINYINT(1) NOT NULL,
  s TINYINT(1) NOT NULL,
  c TINYINT(1) NOT NULL,
  pattern TINYINT(1) NOT NULL
) ENGINE=MyISAM;

CREATE TABLE patterns (
  id TINYINT PRIMARY KEY,
  name VARCHAR(30) NOT NULL,
  emotions VARCHAR(255) NOT NULL,
  goal VARCHAR(255) NOT NULL,
  judges_others VARCHAR(255) NOT NULL,
  influences_others VARCHAR(255) NOT NULL,
  organization_value VARCHAR(255) NOT NULL,
  overuses VARCHAR(255) NOT NULL,
  under_pressure VARCHAR(255) NOT NULL,
  fear VARCHAR(255) NOT NULL,
  effectiveness VARCHAR(255) NOT NULL,
  description TEXT NOT NULL
) ENGINE=MyISAM;
```

Notes:
- The included data uses placeholder “termX/nameX” values; production content is proprietary per the original README.
- Engine is MyISAM; prefer InnoDB in modern environments.

### Legacy setup (to run as-is)
- Requirements: PHP 7.4+ (mysqli), MySQL/MariaDB
- Steps:
  - Create DB `test` and import `db/disc.sql`
  - Update DB creds in `index.php` and `result.php` (`$dbhost`, `$dbuser`, `$dbpass`, `$dbname`)
  - Serve the folder via Apache/Nginx/php-fpm and open `index.php`

### Business rules
- One “Most” and one “Least” per group (28 groups total).
- The app counts D/I/S/C occurrences and calculates “Change” = Most − Least per dimension.
- Segment lookup: for each dimension, find `results.segment` where `graph=3 & dimension=X & value=change`.
- Pattern mapping: find `(d,i,s,c) → pattern` in `pattern_map`, then show `patterns` row.

---

## Modern Rebuild Plan (2025-ready)

### Goals
- Keep scoring logic 1:1 with legacy.
- Improve UX, i18n, security, and deployment.
- Use a modern full-stack with TypeScript, server components, migrations, and tests.

### Recommended stack
- Web framework: Next.js 15 (App Router, Server Actions, RSC)
- Language: TypeScript
- UI: Tailwind CSS (+ headless UI or shadcn/ui)
- ORM: Prisma
- DB: PostgreSQL (Neon/Railway) or MySQL (PlanetScale); either works
- Validation: Zod
- Testing: Vitest (unit), Playwright (e2e)
- Infra: Docker + Compose for local; Vercel/Netlify/Fly/Render for prod

### App architecture
- Routes
  - `/` — renders the 28×4 terms grid, posts selections
  - `/result` — displays final pattern and details
- Actions/API
  - Server Action `submitDiscForm(formData)` performs tally + DB lookups; redirects to `/result` with result id or data
- Data access
  - Read-only tables for `results` and `pattern_map`
  - `personalities` and `patterns` managed by admins or seeded at deploy

### Data schema (Prisma models)
```prisma
model PersonalityTerm {
  id        Int     @id @default(autoincrement())
  groupNo   Int
  term      String
  mostMark  String  @db.Char(1) // 'D','I','S','C','#'
  leastMark String  @db.Char(1)
  @@index([groupNo])
}

model ResultMap {
  id        Int     @id @default(autoincrement())
  dimension String  @db.Char(1) // D/I/S/C
  intensity Int
  value     Int     // change (Most-Least)
  segment   Int
  graph     Int     // uses 3
  @@index([dimension, graph, value])
}

model PatternMap {
  d       Int
  i       Int
  s       Int
  c       Int
  pattern Int
  @@id([d,i,s,c])
  @@index([pattern])
}

model Pattern {
  id                 Int    @id
  name               String
  emotions           String
  goal               String
  judgesOthers       String
  influencesOthers   String
  organizationValue  String
  overuses           String
  underPressure      String
  fear               String
  effectiveness      String
  description        String
}
```

Optional i18n:
- `TermTranslation(termId, locale, text)`
- `PatternTranslation(patternId, locale, name, description, ...)`

### Scoring logic (TypeScript, Server Action)
```ts
// types
type Letter = 'D'|'I'|'S'|'C'|'#';
type Tallies = Record<Exclude<Letter,'#'>, number>;

function tallySelections(most: Letter[], least: Letter[]) {
  const dimensions: Exclude<Letter,'#'>[] = ['D','I','S','C'];
  const init: Tallies = { D:0, I:0, S:0, C:0 };
  const count = (arr: Letter[]) =>
    arr.reduce((acc, cur) => (cur !== '#' ? (acc[cur as keyof Tallies]++, acc) : acc), { ...init });

  const mostCounts = count(most);
  const leastCounts = count(least);

  const change: Tallies = {
    D: mostCounts.D - leastCounts.D,
    I: mostCounts.I - leastCounts.I,
    S: mostCounts.S - leastCounts.S,
    C: mostCounts.C - leastCounts.C
  };

  return { mostCounts, leastCounts, change };
}

// Persisted lookup (Graph III)
async function computePatternId(prisma, change: Tallies) {
  const d = (await prisma.resultMap.findFirst({ where: { graph: 3, dimension: 'D', value: change.D } }))?.segment;
  const i = (await prisma.resultMap.findFirst({ where: { graph: 3, dimension: 'I', value: change.I } }))?.segment;
  const s = (await prisma.resultMap.findFirst({ where: { graph: 3, dimension: 'S', value: change.S } }))?.segment;
  const c = (await prisma.resultMap.findFirst({ where: { graph: 3, dimension: 'C', value: change.C } }))?.segment;
  if (d == null || i == null || s == null || c == null) throw new Error('Segment not found');

  const patternRow = await prisma.patternMap.findUnique({ where: { d_i_s_c: { d, i, s, c } } });
  if (!patternRow) throw new Error('Pattern not found');
  return patternRow.pattern;
}
```

### UI/UX notes
- Use a responsive grid to present 28 groups with sticky column headers.
- Form validation: require one “Most” and one “Least” per group; optionally prevent picking the same term for both.
- Accessibility: label each control, ensure keyboard navigation, high contrast.

### Seeding data
- Convert `db/disc.sql` to JSON or use a small Node script to ingest directly via Prisma.
- Remember: repository terms/patterns are placeholders; replace with your licensed content if needed.

### Local development
- `.env`
  - `DATABASE_URL=postgresql://user:pass@localhost:5432/disc`
- Commands
  - `npm run dev` — Next.js dev server
  - `npx prisma migrate dev`
  - `npx prisma db seed`
  - `npm run test`, `npm run e2e`

### Deployment
- Vercel + Neon (Postgres) or PlanetScale (MySQL)
- Configure `DATABASE_URL` in env
- Run migrations/seed during deployment (or one-off job)
- Add caching headers for static assets and disable caching for form post route

### Testing
- Unit test the scoring function with known inputs/outputs
- E2E: fill the form, submit, assert pattern name renders

### Security and privacy
- Server Actions or API routes validate inputs with Zod
- CSRF protection (built-in for Server Actions in current Next.js)
- Use parameterized queries (Prisma does by default)
- No PII stored; if you store responses, comply with privacy policy and retention limits

### Migration checklist (legacy → modern)
- Export schema/data from `db/disc.sql`
- Map to Prisma models; run migrations
- Seed `personalities`, `results`, `pattern_map`, `patterns`
- Port scoring logic (Most/Least → Change → Segments → Pattern)
- Recreate UI with Tailwind
- Add i18n if needed
- Add tests
- Containerize and deploy

### License and credits
- License: MIT (see `LICENSE`)
- Original author: CAHYA DSN; contributors listed in `README.md`
- This rebuild plan preserves the original behavior while modernizing the stack

---

If you want, I can generate a ready-to-run Next.js app scaffold with Prisma models, seed script, server action, and the test UI. Say “create the modern app” and I’ll add it into a `modern/` folder in this repo.