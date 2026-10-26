# CLAUDE.md

Day.js is a 2 KB immutable date library with a Moment.js-compatible API. The core is kept tiny on purpose (`size-limit` fails the build if `dayjs.min.js` gzips over 2.99 KB), so everything else is an opt-in plugin or locale. The default branch and PR target is `dev`; `master` only receives dev-to-master merges, which cut releases.

## Architecture

- `src/index.js` is the entire core: the `Dayjs` class, the `dayjs()` factory, `extend`, `locale`, and the global locale store `Ls`. Instance state is in `$`-prefixed fields (`$d` is the Date, `$y`..`$ms` its parts, `$L` the locale name); `$x` is the bag for per-instance plugin state.
- `src/utils.js` exports one-letter helpers (`s` padStart, `z` padZoneStr, `m` monthDiff, `a` absFloor, `p` prettyUnit, `u` isUndefined; `index.js` adds `l`, `i`, `w`). Plugins reach them through `this.$utils()`, never by importing `utils.js`.
- `src/plugin/<name>/index.js`: one default export `(option, dayjsClass, dayjsFactory) => {}` that assigns `function`s onto `dayjsClass.prototype` (not arrows, they need `this`). To change an existing method: `const oldX = proto.x; proto.x = function (...) { ...; return oldX.bind(this)(...) }`. Nothing enforces install order; `weekYear` calls `this.week()` from weekOfYear and assumes it is there.
- `src/locale/<code>.js`: the first line must be `// Language (Region) [code]`; `build/index.js` regex-parses it into `locale.json`. `test/locale/keys.test.js` checks every locale: filename equals `name` and is lowercase, `formats` has `LT LTS L LL LLL LLLL` (only `l ll lll llll` may be added), `relativeTime` has exactly 13 keys (no `ss`). Locales `import dayjs from 'dayjs'`, which Jest maps to `src/index.js` through `test/__mocks__/dayjs.js`.
- `types/` is hand-written. `types/plugin/<name>.d.ts` must exist for every `src/plugin/<name>`; `test/plugin/declarations.test.js` fails otherwise. Locales need no types.
- `build/index.js` bundles each locale and plugin to UMD, then `dayjs.min.js`, and copies `types/` to the repo root. `npm run babel` writes `esm/`.
- `docs/` is dead: every page is a redirect stub to day.js.org, which lives in a different repo. Nothing here needs a docs edit.

## Commands

```
npm ci                                   # not `npm install`: npm 10 rewrites package-lock.json
npm run lint                             # eslint src/* test/* build/*  (also the pre-commit hook), ~5 s
TZ=UTC npx jest test/plugin/isLeapYear.test.js --coverage=false   # one file, ~1 s; flag silences a 180-row coverage table
TZ=UTC npm test                          # the CI gate, ~10 s: 8 TZ sub-runs, then jest with a lines=100% threshold
npx cross-env BABEL_ENV=build node build # rollup half of `npm run build`, ~15 s, writes locale/ plugin/ dayjs.min.js index.d.ts locale.json
NODE_OPTIONS=--openssl-legacy-provider npm run size   # the 2.99 KB gate; without the flag Node 17+ throws ERR_OSSL_EVP_UNSUPPORTED
```

`TZ=UTC` is not optional: in a US timezone plain `npm test` fails 6 tests (`localizedFormat`, `dayOfYear`, `timezone`) that are green in CI (ubuntu, UTC). `npm run build` fails on Node 17+ in its last step (`&& npm run size`) for the OpenSSL reason above; `release.yml` sets the same `NODE_OPTIONS`.

## Conventions

- airbnb-base, no semicolons, no trailing commas, single quotes, 2-space indent, imports without `.js`. `npm run lint` is the authority. Nothing lints or type-checks `types/`.
- Tests: `test/<area>.test.js`, `test/plugin/<name>.test.js`, `test/locale/<code>.test.js`, `test/issues/issue<N>.<variant>.test.js`. Each imports `dayjs from '../src'` (or `'../../src'`), calls `dayjs.extend(plugin)` at module top level, and has `beforeEach(() => MockDate.set(new Date()))` / `afterEach(() => MockDate.reset())`. Where moment has the same API it is the oracle: `expect(dayjs(d).format(f)).toBe(moment(d).format(f))`.
- The final jest run enforces 100% line coverage of `src/**`. A new branch without a test fails `npm test`.
- Core uses `arguments`/`.apply` rather than rest/spread, and no optional chaining or `??`. Match that in `src/index.js`; plugins already use spread.
- New plugin = `src/plugin/X/index.js` + `types/plugin/X.d.ts` (`import { PluginFunc } from 'dayjs'` / `declare const plugin: PluginFunc` / `export = plugin` / `declare module 'dayjs' { interface Dayjs { x(): T } }`) + `test/plugin/X.test.js`. `build/` needs no change; it readdirs.
- New locale = `src/locale/xx.js` only; `keys.test.js` validates it. A `test/locale/xx.test.js` is optional (41 of 143 have one).
- Commits use semantic-release prefixes, in practice `fix:` and `chore:`; `CHANGELOG.md` and the version are generated from them.

## Guardrails

- After any build, `plugin/isLeapYear.js`, `locale/en-gb.js`, `dayjs.min.js`, `index.d.ts`, `locale.json` and `esm/` sit at the repo root next to `src/`. They are gitignored output. A search hit there means the real file is under `src/` or `types/`.
- Do not add to `src/index.js` what could be a plugin; `npm run size` is the check.
- Do not "fix" the 6 timezone-dependent failures. Run with `TZ=UTC`.
- Do not run `npm install`, `npm audit fix`, or upgrade devDependencies (babel 7 beta, jest 22, eslint 4, TypeScript 2.8, all from 2018). If `package-lock.json` shows as modified, `git checkout package-lock.json`.
- Do not touch `CHANGELOG.md`, `.releaserc`, `patches/` (patch-package for the release tooling), `karma.sauce.conf.js`, or `test/browser.spec.js` (header says do not modify; it runs only on Sauce Labs via `npm run test:sauce`).
