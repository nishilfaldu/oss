# oss

The site at [oss.nishilfaldu.site](https://oss.nishilfaldu.site) shows my pull requests to Vercel's open-source projects:

- [vercel/streamdown#685](https://github.com/vercel/streamdown/pull/685) `fix(remend): complete links only in the current block`. A stray `[` in an earlier paragraph no longer adds junk to later text, or hides it.
- [vercel/streamdown#689](https://github.com/vercel/streamdown/pull/689) `perf(streamdown): repair only the end of a streaming message`. Each update repairs only the last two blocks. It is stacked on [#670](https://github.com/vercel/streamdown/pull/670) and #685.
- [vercel/workflow#4932](https://github.com/vercel/workflow/pull/4932) `[world-local] Finish step and wait writes that a crash cut short`. A process killed between a step or wait claim and its event no longer leaves the run stuck. The next write takes over the dead writer's claim and writes the missing event.

`prs/` holds each pull request's description and patch.

## The site

A static page with no build step: `index.html`, `styles.css` and `app.js`. The Streamdown demos run the real code from `vendor/`:

| File | Built from |
| --- | --- |
| `remend-main.js` | `packages/remend/src` at upstream `main` (8b0ca9f) |
| `remend-pr685.js` | `packages/remend/src` at #685 (61ee07e) |
| `tail-repair.js` | `repairStream`, `advanceStableLength` and `parseMarkdownIntoBlocks` from `packages/streamdown`, with that branch's remend, at #689 (43c6616) |
| `marked.js` | marked 18.0.11 |

Each file is an esbuild bundle (`--bundle --format=esm --minify`). The Workflow demo is a script of the writes in #4932 (45dd33d), not its code, because the Local World needs a file system and processes. Its numbers come from the pull request's crash test.

`og.png` is the link preview image. `robots.txt`, `sitemap.xml` and the icon files sit next to the page.

To preview, serve the folder, for example `python3 -m http.server`.
