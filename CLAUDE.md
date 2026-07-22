# CLAUDE.md

Project-level instructions for Claude Code. These apply to all contributors working on a clone, regardless of personal config.

## Writing conventions

These apply to everything Claude produces in this project: code, comments, docstrings, markdown, log entries, chat output. They duplicate the user-level preferences at `~/.claude/CLAUDE.md` so the rules travel with the repository and apply even to anyone working on a clone with a different personal config.

**Language:** American English by default (color, behavior, analyze, -ize suffixes). Match the literal spelling of any external library, API, or quoted material that uses other variants - call `analyseSignal()` if that is the library's actual name, even though the prose spelling would be `analyze`.

**Units:** Metric / SI by default (meters, grams, liters, degrees Celsius, hertz, pascals, watts). Exceptions where metric would cause confusion: imperial-only standards (screw threads like "1/4-20", AWG wire gauges, pipe diameters in inches), datasheets that use imperial, historical material, or direct quotations. Quote the original unit and add a metric conversion in commentary when it helps. Encode units in variable name suffixes (`length_mm`, `period_ms`, `voltage_mV`) when unit-of-measure matters.

**Characters in code, code comments, and docstrings: strict ASCII only.**

- No long dashes (em or en); use hyphen `-` or double hyphen `--`.
- No Unicode arrows; use `->` in prose, language operators in code.
- No superscript or subscript characters; spell them out (`**2`, `^2`, "squared"; `_i` or `[i]` for indexing).
- No Greek letters; spell them out (`alpha`, `beta`, `mu`, `theta`).
- No mathematical operator glyphs (integral, sigma, etc.); spell them out (`sum`, `integral_of`).
- No typographic quotes; use straight `"` and `'`.
- No ellipsis character; use three dots `...`.
- No Czech (or other) diacritics in identifiers, comments, or docstrings; use the diacritic-free form.

String literals that must contain real-world data (file paths, user-facing strings in other languages) are exempt - this rule is about authoring style, not data integrity.

**Characters in markdown, chat, and config files: ASCII plus Czech diacritics.**

The Czech alphabet is permitted here: á č ď é ě í ň ó ř š ť ú ů ý ž and the uppercase forms (Á Č Ď É Ě Í Ň Ó Ř Š Ť Ú Ů Ý Ž). Still avoid em/en dashes, Unicode arrows, curly quotes, ellipsis character, multiplication sign, non-breaking spaces, and other typographic glyphs.

**Decorations in code comments and docstrings: none.**

No long lines of `-`, `=`, `*`, or `#` as separators. No boxed banners around section headers. A single blank line is enough as a visual break.

Exception: when a documentation format (reStructuredText, JSDoc, Sphinx, etc.) requires specific markers for parsing, use exactly what the format mandates and no more.
