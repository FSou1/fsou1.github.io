---
title: "2023 - Now, PayPal"
order: 8
sidebar:
  - title: "Role"
    text: "Staff Software Engineer (Frontend Architect)"
  - title: "Tech stack"
    text: "React, TypeScript, Storybook, Node.js, Turborepo"
  - title: "Location"
    text: "Houston, TX"
  - title: "Website"
    text: "[https://www.paypal.com/](https://www.paypal.com/)"
---

As a Staff Software Engineer / Frontend Architect at PayPal, I am a core technical contributor and tech lead on the PPCOM team (11 engineers). We own the React component platform that powers PayPal.com and Venmo.com: more than 6,500 pages and 4.8M daily views. I lead the architecture design, standards, and technical direction for the foundations that all component work builds on.

My biggest achievements include improving PayPal.com homepage LCP by 36% on desktop and 46% on mobile, migrating Venmo.com from a static site to the component platform end-to-end, and building the tokenization and multi-brand theming foundation behind the Oslo rebrand. I also re-architected the component library's CI/CD into a package-agnostic monorepo pipeline, and I am now driving AI tooling adoption across the platform to shorten the release cycle.

## My contribution

- Lead the architecture design, standards, and technical direction for the React component platform powering PayPal.com and Venmo.com (6,500+ pages, 4.8M daily views), as tech lead on the PPCOM team of 11 engineers.
- Improved PayPal.com homepage LCP by 36% on desktop (3.48s → 2.24s) and 46% on mobile (4.24s → 2.29s) through a component platform redesign and rendering optimizations, shipped via controlled rollouts with zero incidents and full backward compatibility.
- Led the Venmo.com migration from a static site to the component platform end-to-end: migrated all public marketing and legal pages (260+), cut page development from days to hours, and delivered six-figure annual savings by retiring Contentful, while enabling A/B testing and no-code launches.
- Built the tokenization and multi-brand theming foundation: a Tokens Studio → Style Dictionary pipeline with CI/release automation that cut theme foundation work from weeks to days and enabled QuantumLeap, PayPal Everywhere, Fastlane, and the Oslo rebrand (5,000+ pages, 201 markets) with zero breaking changes.
- Re-architected the component library's CI/CD, moving off a monolithic Jenkins pipeline to a package-agnostic monorepo on a Turbo-driven Harness pipeline that scales build, test, and release across every package, so adding a new package requires zero pipeline changes.
- Made visual regression testing trustworthy at scale: about 3,400 Playwright screenshots of Storybook stories across four browsers and multiple brand themes, generated only in CI with fonts and media served from the repo, so a failure means something actually changed ([how we keep visual tests reliable]({% post_url 2026-11-12-Visual_testing_at_paypal %})).
- Designed a multi-library architecture on that monorepo, letting teams ship short-lived, experimental marketing experiences in isolated libraries that protect page performance and retire without leaving debt in the core design system.
- Driving AI tooling adoption across the platform: Storybook MCP for automated release sanity checks, AI-assisted design-to-code workflows replacing static Figma handoffs, and custom Claude skills for developer environment workflows, to shorten the release cycle and speed up component delivery.
- Built performance governance and release tooling, including a bundle size analyzer integrated into PR reviews and CI/CD workflows, and supported the Jest to Vitest migration.
- Contributed hands-on to component development with a strong focus on accessibility, responsiveness, maintainability, and production quality.

## Skills

- **Leadership:** Technical Leadership, Frontend Architecture, Architecture Reviews, Cross-team Alignment
- **Frontend:** React, TypeScript, JavaScript, Node.js, CSS / LESS
- **Design Systems:** Component Libraries, Storybook, Design Tokens (Tokens Studio, Style Dictionary), Multi-brand Theming, Accessibility (WCAG / ARIA)
- **Performance:** Core Web Vitals, Rendering Optimization, Bundle Size Governance
- **Platform & Delivery:** Monorepos (Turborepo), CI/CD (Harness, Jenkins), Release Automation, Large-scale Migrations
- **Quality:** Testing Strategy, Visual Regression (Playwright), E2E Testing, Vitest / Jest
- **AI Engineering:** AI-assisted Development, MCP, Claude Code Skills, Design-to-Code Workflows
