# Road Trip Expense Calculator

> Estimate before you go, so you don't overspend on the road.

A cross-platform road trip expense estimator built with **uni-app (Vue 3)**. Enter your itinerary, vehicle energy consumption, lodging and dining, plus extra budget items — then generate a one-tap expense report, view the cost breakdown as a rose chart, and share it as a poster or a report link.

English | [中文](./README.md)

## Features

### Trip Input (Home)

- **Basic trip**: driving distance (one way), round trip toggle (total mileage = one-way × 2), whether the highway is used.
- **Vehicle type & energy consumption**: switch between gasoline / electric / hybrid vehicles.
  - Gasoline: fuel consumption per 100 km, current fuel price.
  - Electric: energy consumption per 100 km, average electricity price.
  - Hybrid: energy consumption per 100 km, average electricity price, fuel consumption when running on fuel, current fuel price, plus an **EV ratio** slider to weight the two energy sources.
- **Trip & travelers**: trip duration (days), number of travelers.
- **Lodging & dining**: average lodging cost per day, dining cost per person per day.
- **Extra budget**: freely add or remove items, and mark whether an item is calculated **per person**.

### Expense Report

- **Summary**: total cost, cost per km, cost per person, total mileage.
- **Cost breakdown**: rose chart (uCharts) of category proportions, with a color legend.
- **Line items**: amount, percentage, and a proportion bar for each category.
- **Extra spending details**: each item with its amount and a "per person / total" tag.
- **Trip summary**: vehicle type · days · travelers · total mileage.

### Sharing

- **Image share**: renders a share poster offscreen with the Canvas 2D API, which can be saved to the photo album.
- **Report share** (mini program): the share carries compressed report data so a friend opens the exact same report.

### Experience Details

- **Dark mode**: automatically follows the system `prefers-color-scheme`, with navigation bar colors managed by `theme.json`.
- **Zero-amount categories are hidden**: e.g. the highway fee is not shown when no highway is used.
- **When there is only 1 traveler**: the cost-per-person card is hidden and the total cost card spans the full row.

## Tech Stack

| Category | Choice |
| --- | --- |
| Framework | uni-app (Vue 3) |
| UI library | [uni-ui](https://github.com/dcloudio/uni-ui) 1.5.6 |
| Charts | [qiun-data-charts](https://gitee.com/uCharts/uCharts) (uCharts) 2.5.0 |
| Poster rendering | Mini program / H5 Canvas 2D API |
| Theming | CSS variables + `theme.json` dark mode config |

## Project Structure

```
rtc/
├── App.vue                      # App entry, global theme (CSS variables) and dark mode
├── main.js                      # App bootstrap (Vue 3 createSSRApp)
├── pages.json                   # Page routes and navigation bar config
├── manifest.json                # App config (App / mini program appids, etc.)
├── theme.json                   # Light/dark theme variables (nav bar, page background)
├── uni.scss                     # uni-app built-in style variables
├── index.html                   # H5 template
├── pages/
│   ├── index/index.vue          # Home: trip and budget input
│   └── report/report.vue        # Report: summary, chart, details, sharing
├── static/
│   ├── minicode.png             # WeChat mini program code (for the poster)
│   └── dycode.png               # Douyin mini program code (for the poster)
└── uni_modules/                 # uni-ui and qiun-data-charts components
```

## Getting Started

This is an HBuilderX project (no CLI), so opening it with HBuilderX is recommended.

1. Open the project root with [HBuilderX](https://www.dcloud.io/hbuilderx.html).
2. Choose **Run → Run to Browser / Run to Mini Program Simulator** (e.g. WeChat DevTools).
3. To run on WeChat Mini Program, replace `mp-weixin.appid` in `manifest.json` with your own AppID.

> When running in WeChat DevTools for the first time, enable the "Service Port" in the tool so HBuilderX can connect.

## Scan to Try

Scan the mini program code below with the matching app to try it out (the images live in [`static/`](./static) and are also used as the QR code in the share poster).

| WeChat Mini Program | Douyin Mini Program |
| --- | --- |
| <img src="./static/minicode.png" width="200" alt="WeChat mini program code" /> | <img src="./static/dycode.png" width="200" alt="Douyin mini program code" /> |

## Calculation Rules

| Item | Formula |
| --- | --- |
| Total mileage | `one-way distance × (round trip ? 2 : 1)` |
| Driving cost (gasoline) | `total mileage × fuel consumption per 100 km ÷ 100 × fuel price` |
| Driving cost (electric) | `total mileage × energy consumption per 100 km ÷ 100 × electricity price` |
| Driving cost (hybrid) | `total mileage × energy consumption ÷ 100 × electricity price × EV ratio + total mileage × fuel consumption ÷ 100 × fuel price × (1 − EV ratio)` |
| Highway fee | `uses highway ? total mileage × 0.5 CNY/km : 0` |
| Lodging | `lodging cost per day × days` |
| Dining | `dining cost per person per day × travelers × days` |
| Extra spending | `Σ (per person ? amount × travelers : amount)` |
| Total cost | Sum of the five categories above |
| Cost per person | `total cost ÷ travelers` |
| Cost per km | `driving cost ÷ total mileage` |

### Defaults

| Parameter | Default |
| --- | --- |
| Mileage / days / travelers | 500 km / 5 days / 2 people |
| Lodging / dining | 300 CNY/day · 100 CNY/person/day |
| Gasoline | 8 L/100km · 8 CNY/L |
| Electric | 15 kWh/100km · 1.5 CNY/kWh |
| Hybrid | 15 kWh/100km · 1.5 CNY/kWh · 5 L/100km · 8 CNY/L · EV ratio 50% |
| Extra budget | Parking 100 CNY (total), attraction tickets 100 CNY (per person) |

## Category Colors

To keep the chart, legend, and line items consistent, the five categories use fixed colors:

| Category | Color |
| --- | --- |
| Driving cost | `#3b82f6` blue |
| Highway fee | `#f59e0b` amber |
| Lodging | `#10b981` emerald |
| Dining | `#ec4899` pink |
| Extra spending | `#8b5cf6` violet |

## Platform Support

| Platform | Status | Notes |
| --- | --- | --- |
| WeChat Mini Program | ✅ | Report share, poster generation and save-to-album |
| Douyin Mini Program | ✅ | Uses `dycode.png` as the poster QR code |
| H5 | ✅ | Rose chart and poster generation supported |
| App (Android / iOS) | ✅ | Configured via `manifest.json` |
| Alipay / Baidu / QQ Mini Program | ⚠️ | Basic config only, no dedicated adaptation |

## Contributing

Issues and pull requests are welcome. Before submitting, make sure all platforms run correctly and that category colors and calculation rules stay consistent.

## License

This project is licensed under the [MIT License](./LICENSE).

Copyright (c) 2026 Willin Wang
