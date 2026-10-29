# Crawdad 2026 (v2.0.1)
Centering Resonance Analysis (CRA) of text, as a stand-alone web application.
Runs entirely in your browser — no installation, no server, no account, and
**no data leaves your computer**.

## How to run

1. Unzip the folder anywhere on your computer.
2. Double-click `index.html` (or drag it into any modern browser: Chrome, Edge,
   Firefox, Safari).
3. That's it. The app works fully offline.

Or host the folder on any static web server (e.g., GitHub Pages) — no build
step or configuration needed.

## What it does

- **Generator** — turns text files (.txt, .md, .pdf, .docx) into CRA network
  files (`*_cra.md`): noun-phrase parsing, linking, and influence
  (betweenness centrality) per Corman, Kuhn, McPhee, & Dooley (2002).
- **Visualizer** — draws the CRA network; exports PowerPoint-ready PNGs.
- **Comparator** — word and pair resonance between two texts (Eqs. 2–6).
- **Classifier** — clusters texts by mutual resonance (K-means, √n rule, MDS plot).
- **Theme Identifier** — manifest themes via PCA with varimax rotation.
- **Latent Theme Identifier** — scores texts against your own theme dictionary.
- **Summarizer** — most influential words across a corpus.

## Documentation

The full guide is in this folder: **Crawdad Users Manual v2.0.1.docx** —
installation, every module, the text-metrics reference, and troubleshooting.

## Verify your copy

- Open `tests/index.html` and run each suite — every page should show all PASS.
  (The same suites run in CI-style via Node: `node tests/<name>.js`.)
- `CHECKSUMS.txt` lists the SHA-256 hash of every file. To verify on macOS:
  `shasum -a 256 -c CHECKSUMS.txt` from inside this folder
  (Linux: `sha256sum -c CHECKSUMS.txt`).
- See `SECURITY-AND-PROVENANCE.md` for the security properties of this app and
  the provenance and licenses of the bundled libraries.

## License

Copyright 2026 Kevin J. Dooley. Licensed under the **Apache License,
Version 2.0** — see `LICENSE`. Attribution notices are in `NOTICE`;
bundled third-party components and their licenses are listed in
`THIRD-PARTY-NOTICES.md`. First-party source files carry an
`SPDX-License-Identifier: Apache-2.0` header; files under `vendor/` are
third-party and keep their own licenses.

## Citation

Please cite: Corman, S., Kuhn, T., McPhee, R., & Dooley, K. (2002). Studying
complex discursive systems: Centering Resonance Analysis of organizational
communication. *Human Communication Research, 28*(2), 157–206.
