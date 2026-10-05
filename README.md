# Project Yapbook

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   ██████╗ ██████╗  ██████╗      ██╗███████╗ ██████╗████████╗           │
│   ██╔══██╗██╔══██╗██╔═══██╗     ██║██╔════╝██╔════╝╚══██╔══╝           │
│   ██████╔╝██████╔╝██║   ██║     ██║█████╗  ██║        ██║              │
│   ██╔═══╝ ██╔══██╗██║   ██║██   ██║██╔══╝  ██║        ██║              │
│   ██║     ██║  ██║╚██████╔╝╚█████╔╝███████╗╚██████╗   ██║              │
│   ╚═╝     ╚═╝  ╚═╝ ╚═════╝  ╚════╝ ╚══════╝ ╚═════╝   ╚═╝              │
│                                                                          │
│              projects, experiments, and yaps - indexed                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

A public memory index for Kao's projects. Each project has its own folder with a short description and logo. Open a project card from the table below.

> The index stores lightweight notes and branding only. It does **not** copy project source trees, `.env` files, credentials, dependencies, or build output.

## Projects

| Logo | Project | Short description |
|---|---|---|
| ![SkautReg logo](./SkautReg/logo.svg) | **[SkautReg](SkautReg/)** | Scout group management for members, trips, RSVP, meetings, transport, finance, and integrations. |
| ![Alplight Share logo](./alplight-share/logo.svg) | **[Alplight Share](alplight-share/)** | A live-folder workspace for encrypted uploads, connected computer sources, WebDAV drives, and lightweight media proxies. |
| ![stamp.dev logo](./stampdotdev/logo.svg) | **[stamp.dev](stampdotdev/)** | A speed-first Git forge prototype with Git-backed storage, smart HTTP transport, a web UI, and benchmarking tools. |
| ![Spilled logo](./spilled/logo.svg) | **[Spilled](spilled/)** | The local Spilled media-library workspace: saved library state, offline records, and the local content vault used alongside SpilledCinema. |
| ![SpilledCinema logo](./cinema/logo.png) | **[SpilledCinema](cinema/)** | A self-hosted media streaming platform with local nodes, a dashboard, gateway, control plane, desktop shell, and browser bridge. |
| ![kaoID logo](./kaoid/logo.svg) | **[kaoID](kaoid/)** | Shared identity and authentication infrastructure using short-lived JWTs, JWKS verification, and session-aware app access. |
| ![quickHOST logo](./quickhost/logo.svg) | **[quickHOST](quickhost/)** | One-command static hosting for Vite, Astro, and HTML apps on kaooffline.top with Cloudflare, D1/R2 storage, CLI deployment, and 30-day expiry. |
| ![FlashGames logo](./flashgames/logo.svg) | **[FlashGames](flashgames/)** | An outdoor multiplayer game platform foundation using Convex and Convex Auth, with FlashTag as its first game mode. |
| ![myamipisnicky logo](./myamipisnicky/logo.svg) | **[myamipisnicky](myamipisnicky/)** | A Czech and Slovak song-lyrics app with a deterministic crawler, SQLite index, local search, and a Vinext/Cloudflare web starter. |
| ![FactCheck logo](./factcheck/logo.png) | **[FactCheck](factcheck/)** | A source-grounded fact-checking web app that plans research, compares evidence, and returns citation-checked verdicts. |
| ![VEB logo](./veb/logo.svg) | **[VEB](veb/)** | The Universal Runtime: a cross-platform tool for installing and running complete applications from VEXP project manifests. |
| ![SkautIS Wrapper logo](./skautis-wrapper/logo.svg) | **[SkautIS Wrapper](skautis-wrapper/)** | A Fastify gateway with per-session queues and isolated Chromium workers for SkautIS, plus a dashboard and navigation map. |
| ![Beam CMS logo](./beam/logo.svg) | **[Beam CMS](beam/)** | A CMS concept that combines visually managed content and hash-verified publishing with the speed of a static site. |
| ![TheRaceTeam logo](./raceteam/logo.svg) | **[TheRaceTeam](raceteam/)** | A race-management MVP for theraceteam.cz with public race pages, organizer tools, scheduling, and capacity settings. |
| ![slimySOFTWARE logo](./slimy-software/logo.png) | **[slimySOFTWARE](slimy-software/)** | A Vite/React software catalog interface for browsing installer packages, deployment profiles, and app metadata. |

## Folder map

```text
Project Yapbook/
├── SkautReg/
├── alplight-share/
├── stampdotdev/
├── spilled/
├── cinema/
├── kaoid/
├── quickhost/
├── flashgames/
├── myamipisnicky/
├── factcheck/
├── veb/
├── skautis-wrapper/
├── beam/
├── raceteam/
└── slimy-software/
```

## Notes

- Descriptions were assembled from the project READMEs, package metadata, and app source found on the local computer.
- `spilled` is represented as the local media-library workspace; `cinema` is the SpilledCinema application monorepo.
- `kaoID` is represented as a shared identity layer because the local computer contains its implementation and branding inside related projects rather than as a separate checkout.
