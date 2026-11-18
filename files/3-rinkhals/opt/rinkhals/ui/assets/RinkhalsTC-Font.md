# Rinkhals TC font

`RinkhalsTC-Regular.ttf` is a static weight-400 subset of [Noto Sans TC](https://github.com/google/fonts/tree/3be1884c48c3e45b52ecc725676a08f87776373e/ofl/notosanstc). The upstream variable TrueType file has SHA-256 `864727d210d54f2537bbe23b3a839436c3992af72de9322af5270897246bd44f`.

To refresh the subset, instantiate weight 400 with FontTools and keep the CJK characters and fullwidth punctuation used in `ui/*.py` and the bundled `app.json` manifests, plus U+0020–U+024F, U+2000–U+206F, and U+2192. Rename the derived family's name records to Rinkhals TC. The bundled icon font is separate and unchanged.

The subset is intentionally small for the touchscreen environment. It is not a general-purpose font for arbitrary Chinese filenames or third-party text. See `RinkhalsTC-OFL.txt` for the license and attribution.
