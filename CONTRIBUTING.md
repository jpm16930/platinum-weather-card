# Contributing

## Building

```bash
npm install
npm run rollup
```

The build outputs a single file: `dist/platinum-weather-card.js`.
All code (including the card configuration editor) is inlined into this one file —
there are no separate chunk or editor files to worry about.

## After building

Commit the updated file and push:

```bash
git add dist/platinum-weather-card.js
git commit -m "Build: update dist bundle"
git push origin master
```

Then update the card via HACS in Home Assistant.

## Versioning

The card version is set in `src/const.ts`. Update `CARD_VERSION` before releasing.

## Upstream

This fork is based on [Makin-Things/platinum-weather-card](https://github.com/Makin-Things/platinum-weather-card).
To pull in upstream fixes:

```bash
git fetch upstream
git merge upstream/master
```
