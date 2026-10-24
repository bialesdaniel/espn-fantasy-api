# Documentation Development Guide

This directory contains the OpenAPI 3.0 specification for the ESPN Fantasy API
and the static [Swagger UI](https://github.com/swagger-api/swagger-ui) frontend
used to view it. The site is deployed to GitHub Pages at
[https://bialesdaniel.github.io/espn-fantasy-api](https://bialesdaniel.github.io/espn-fantasy-api).

---

## Directory Overview

```text
docs/
├── index.html              # Main HTML entry point loading Swagger UI
├── openapi/                # OpenAPI 3.0 specification files (split for modularity)
│   ├── open-api.yml        # Root OpenAPI specification and endpoint paths
│   ├── shared-components.yml # Common schemas shared across multiple views/endpoints
│   ├── no-view-league.yml  # Schema for default league response (no view parameter)
│   ├── mteam-league.yml    # Schema for league response with view=mTeam
│   └── mroster-league.yml  # Schema for league response with view=mRoster
├── ui/                     # Pre-built Swagger UI distribution assets (CSS, JS)
│   └── swagger-initializer.js # Configures Swagger UI to load openapi/open-api.yml
├── swagger-ui.version      # Tracks the currently bundled Swagger UI release tag
└── README.md               # This guide
```

---

## Running Swagger UI Locally

Because Swagger UI fetches the OpenAPI YAML files via `fetch`/`XMLHttpRequest`,
**opening `index.html` directly from your file system (`file://...`) will fail**
due to browser Cross-Origin Resource Sharing (CORS) restrictions.

To view and test changes locally, run a lightweight static HTTP server from the
project:

### Option 1: Python 3 (Recommended)

From the project root:

```bash
python3 -m http.server 8000 --directory docs
```

Or from within the `docs/` directory:

```bash
python3 -m http.server 8000
```

Then navigate to [http://localhost:8000](http://localhost:8000) in your browser.

### Option 2: Node (`npx serve`)

```bash
npx serve docs -p 8000
```

---

## Specification Architecture

The ESPN Fantasy API uses polymorphic responses for the league endpoint
(`/segments/0/leagues/{leagueId}`) depending on the `view` query parameter.

To keep the documentation clean and maintainable, the OpenAPI specification is
split across multiple files using `$ref` pointers:

1. **`openapi/open-api.yml`**:
   - Root document declaring `openapi: 3.0.3`, server URLs, and base path
     definitions.
   - Defines the `view` query parameter enum (`LeagueViews`).
   - Uses `oneOf` and a `discriminator` on the `view` property to map responses
     to view-specific schema files.

2. **`openapi/shared-components.yml`**:
   - Common building blocks reused by multiple responses (e.g.
     `BasicLeagueResponse`, `BasicLeagueMember`, `DraftDetails`,
     `BasketballStats`, `InjuryStatus`).

3. **View-specific response files** (e.g. `no-view-league.yml`,
   `mteam-league.yml`, `mroster-league.yml`):
   - Combine shared components using `allOf` with view-specific additions.

---

## Workflow: Adding or Modifying Endpoints and Views

### Adding a New League View

The ESPN API supports several additional `view` query parameters (such as
`mMatchup`, `mSettings`, `mStandings`). To document a new view:

1. **Create the View Schema File**:
   - Add `docs/openapi/<view-name>-league.yml`.
   - Inherit common properties using `allOf` pointing to
     `shared-components.yml#/components/schemas/BasicLeagueResponse`.
   - Define view-specific schemas within the new file.

2. **Add Shared Components (if reusable)**:
   - If models introduced by the new view are shared by other views or
     endpoints, add them to `docs/openapi/shared-components.yml`.

3. **Register the View in `open-api.yml`**:
   - Add the view value to the `LeagueViews` enum under
     `components.schemas.LeagueViews`.
   - Add the reference to `responses.200.content.application/json.schema.oneOf`:
     ```yaml
     - $ref: './<view-name>-league.yml#/components/schemas/<ResponseName>'
     ```
   - Add the mapping under `discriminator.mapping`:
     ```yaml
     <view-name>: './<view-name>-league.yml#/components/schemas/<ResponseName>'
     ```

### Adding a New Endpoint

1. In `docs/openapi/open-api.yml`, define the path, HTTP method, parameters, and
   response schemas under `paths:`.
2. If responses are large or distinct, create a dedicated YAML file in
   `docs/openapi/` and reference it via `$ref`.

---

## Validating Changes

Before committing changes, ensure your OpenAPI YAML files are syntactically
valid and all `$ref` pointers resolve correctly:

### 1. In Swagger UI (Browser)

- Reload [http://localhost:8000](http://localhost:8000) after making edits.
- Open the **Browser Developer Tools** (Console and Network tabs).
- Check for 404s on referenced YAML files or Swagger UI parser error modals at
  the top/bottom of the page.

### 2. Command-Line Linting (Optional)

You can validate the OpenAPI spec using standard OpenAPI tools like Redocly CLI
or Spectral:

```bash
# Using Redocly CLI
npx @redocly/cli lint docs/openapi/open-api.yml

# Using Spectral
npx @stoplight/spectral-cli lint docs/openapi/open-api.yml
```

---

## Upstream Swagger UI Updates

Swagger UI assets inside `docs/ui/` are kept up to date automatically via a
GitHub Actions workflow:

- **Workflow File**: `.github/workflows/update-swagger.yml`
- **Schedule**: Runs daily at 10:00 UTC (and supports manual execution via
  `workflow_dispatch`).
- **Mechanism**:
  1. Checks the latest release tag from `swagger-api/swagger-ui`.
  2. Compares against `docs/swagger-ui.version`.
  3. Downloads and extracts the new `dist/` directory into `docs/ui/`.
  4. Patches `ui/swagger-initializer.js` and `docs/index.html` with project
     paths.
  5. Opens an automated pull request with the updated release tag.
