# Contributing

Thank you for considering a contribution to Analisis Pajak Emas. This project is a Vue.js financial analytics dashboard for Ar-Rahnu / pajak emas simulation using historical Bank Negara Malaysia Kijang Emas data.

## Contribution Goals

Contributions should keep the project:

- Accurate and transparent in its financial assumptions.
- Professional enough for portfolio and recruiter review.
- Easy to run locally with standard Vue/Vite tooling.
- Responsive across desktop, tablet, and mobile screens.
- Clear about the difference between simulation and financial advice.

## Getting Started

Fork and clone the repository:

```bash
git clone https://github.com/your-username/Analisis-Pajak-Emas.git
cd Analisis-Pajak-Emas
```

Install dependencies:

```bash
npm install
```

Run the development server:

```bash
npm run dev
```

Build before submitting a pull request:

```bash
npm run build
```

## Recommended Workflow

1. Create a focused branch:

```bash
git checkout -b feature/your-feature-name
```

2. Make a small, clear change.
3. Test the dashboard locally.
4. Run the production build.
5. Submit a pull request with a concise explanation.

## Pull Request Guidelines

Please include:

- A short summary of the change.
- Screenshots for UI changes.
- Any formula or data assumptions changed.
- Testing notes, including `npm run build` result.

For financial logic changes, clearly explain the formula and why the change is needed.

## Code Style

- Follow the existing Vue 3 Composition API style.
- Keep calculations readable and documented when the formula is not obvious.
- Prefer small helper functions for reusable financial logic.
- Keep Bahasa Melayu labels where they improve local context.
- Keep README and documentation in professional English.

## Data Guidelines

- Keep source data traceable.
- Document the origin of any new dataset.
- Avoid modifying historical data without a clear reason.
- Prefer adding data validation or transformation notes when data format changes.

## Financial Disclaimer

This project is for educational, analytics, and portfolio demonstration purposes only. Contributions must not present the simulation as financial advice, investment advice, legal advice, or Shariah advisory.

## Reporting Issues

When opening an issue, include:

- What you expected to happen.
- What actually happened.
- Steps to reproduce.
- Browser and device details if the issue is visual.
- Screenshot or screen recording when useful.
