# Project Structure

This document explains the current repository layout and a suggested structure for scaling the dashboard into a larger fintech analytics project.

## Current Structure

```text
.
├── index.html
├── package.json
├── package-lock.json
├── vite.config.js
└── src
    ├── App.vue
    ├── main.js
    └── data
        ├── asb-dividends.json
        └── kijang-emas-prices.json
```

## Current Folder and File Roles

| Path | Purpose |
| --- | --- |
| `index.html` | Vite HTML entry point. |
| `package.json` | Project metadata, scripts, and dependencies. |
| `package-lock.json` | Locked npm dependency versions for reproducible installs. |
| `vite.config.js` | Vite configuration with Vue plugin registration. |
| `src/main.js` | Vue application bootstrap file. |
| `src/App.vue` | Main dashboard UI, financial calculations, chart setup, and styling. |
| `src/data/asb-dividends.json` | Local ASB dividend and bonus dataset. |
| `src/data/kijang-emas-prices.json` | Local historical BNM Kijang Emas price dataset. |

The current structure is suitable for a compact single-page dashboard. As the project grows, separating calculations, components, and assets will make the codebase easier to maintain.

## Suggested Production Structure

```text
.
├── public
│   └── screenshots
│       ├── dashboard-overview.png
│       ├── chart-section.png
│       ├── restructure-table.png
│       └── mobile-view.png
├── src
│   ├── assets
│   │   └── styles
│   │       └── main.css
│   ├── components
│   │   ├── AppHero.vue
│   │   ├── SimulationControls.vue
│   │   ├── SummaryCards.vue
│   │   ├── CashflowChart.vue
│   │   ├── ComparisonTable.vue
│   │   ├── RestructureTable.vue
│   │   └── PriceDataModal.vue
│   ├── data
│   │   ├── asb-dividends.json
│   │   └── kijang-emas-prices.json
│   ├── utils
│   │   ├── finance.js
│   │   ├── formatters.js
│   │   └── dates.js
│   ├── App.vue
│   └── main.js
├── docs
│   └── formulas.md
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── PROJECT_STRUCTURE.md
├── LICENSE
├── package.json
└── vite.config.js
```

## Suggested Folder Roles

| Path | Purpose |
| --- | --- |
| `public/screenshots/` | Static screenshots used by README and portfolio pages. |
| `src/assets/` | Shared visual assets and global styles. |
| `src/assets/styles/main.css` | Extracted global dashboard CSS. |
| `src/components/` | Reusable Vue components for each dashboard section. |
| `src/data/` | Versioned local datasets used by the simulation. |
| `src/utils/finance.js` | Financial formulas such as gold valuation, pawn proceeds, storage fee, restructure surplus, and compounding. |
| `src/utils/formatters.js` | Currency, number, date, and percentage formatting helpers. |
| `src/utils/dates.js` | Date range, month interval, and price lookup helpers. |
| `docs/` | Additional technical notes, formula references, and screenshot assets. |

## Refactor Recommendation

The next practical cleanup is to split `src/App.vue` into:

- Presentation components for each visual section.
- A finance utility module for reusable formulas.
- A formatter utility module for display helpers.
- A dedicated stylesheet file for global dashboard styles.

This keeps the main application readable and makes the financial logic easier to test.

## Screenshot Recommendation

Add these images before promoting the repository on GitHub or LinkedIn:

```text
docs/screenshots/dashboard-overview.png
docs/screenshots/chart-section.png
docs/screenshots/restructure-table.png
docs/screenshots/mobile-view.png
```

Recommended capture sizes:

| Screenshot | Suggested Size |
| --- | --- |
| Dashboard overview | 1440px desktop viewport |
| Chart section | 1440px desktop viewport |
| Detailed restructure table | 1440px desktop viewport |
| Mobile responsive view | 390px or 430px mobile viewport |

## Repository Presentation Checklist

- Add screenshots and verify all README image paths.
- Add the short repository description in GitHub settings.
- Add the recommended topics in GitHub settings.
- Confirm the Cloudflare Pages URL is visible in the repository About section.
- Keep `dist/` ignored unless a deployment platform specifically requires committed build output.
- Run `npm run build` before publishing major changes.
