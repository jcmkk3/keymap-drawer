# `keymap-drawer`

Parse QMK & ZMK keymaps and draw them in vector graphics (SVG) format, with support for visualizing hold-taps and combos that are commonly used with smaller keyboards.

Available as a [command-line tool](#command-line-tool-installation) or a [web application](https://caksoylar.github.io/keymap-drawer).

![Example keymap](https://caksoylar.github.io/keymap-drawer/3x5.rot.svg)

## Features

- Draw keymap representations consisting of multiple layers, hold-tap keys and combos
  - Uses a human-editable YAML format for specifying the keymap
  - Non-adjacent or 3+ key combos can be visualized by specifying its positioning relative to the keys, with automatically drawn dendrons to keys
- Bootstrap the YAML representation by automatically parsing QMK or ZMK keymap files
- Arbitrary physical keyboard layouts (with rotated keys!) supported, along with parametrized ortho layouts
- Both parsing and drawing are customizable with a config file, see ["Customization" section](#customization)

See examples in [the live web demo](https://caksoylar.github.io/keymap-drawer) for example inputs and outputs.

Compared to to visual editors like [KLE](http://www.keyboard-layout-editor.com/), `keymap-drawer` takes a more programmatic approach.
It also decouples the physical keyboard layout from the keymap (i.e., layer and combo definitions) and provides the tooling to bootstrap it quickly from existing firmware configuration.

## Usage

### Try it as a web application

You can try the keymap parsing and drawing functionalities with a [Streamlit](https://streamlit.io) web application available at https://caksoylar.github.io/keymap-drawer.
Below instructions mostly apply for the web interface, where subcommands and option flags are mapped to different widgets in the UX.

### Command-line tool installation

The recommended way to install `keymap-drawer` is through [pipx](https://pypa.github.io/pipx/), which sets up an isolated environment and installs the application with a single command:

```sh
pipx install keymap-drawer
```

This will make the `keymap` command available in your `PATH` to use:

```sh
keymap --help
```

Alternatively, you can `pip install keymap-drawer` in a virtual environment or install into your user install directory with `pip install --user keymap-drawer`.
See [the development section](#development) for instructions to install from source.

### Bootstrapping your keymap representation

**`keymap parse`** command helps to parse an existing QMK or ZMK keymap file into the keymap YAML representation the `draw` command uses to generate SVGs.
`-c`/`--columns` is an optional parameter that specifies the total number of columns in the keymap to better reorganize output layers.

- **QMK**: Only json-format keymaps are supported, which can be exported from [QMK Configurator](https://config.qmk.fm/), converted from `keymap.c` via `qmk c2json`, or from a VIA backup json via `qmk via2json`:

  ```sh
  qmk c2json ~/qmk_firmware/keyboards/ferris/keymaps/username/keymap.c | keymap parse -c 10 -q - >sweep_keymap.yaml
  ```

  Due to current limitations of the `keymap.json` format, combos and `#define`'d layer names will not be present in the parsing output.

- **ZMK**: `.keymap` files are used for parsing. These will be preprocessed similar to the ZMK build system, so `#define`'s and `#include`s will be expanded.

  ```sh
  keymap parse -c 10 -z ~/zmk-config/config/cradio.keymap >sweep_keymap.yaml
  ```

  Currently combos, hold-taps (including custom ones), layer names and sticky keys (`&sk`/`&sl`) can be determined via parsing.
  For layer names, the value of the `label` property will take precedence over the layer's node name if provided.

  > **Warning**
  >
  > Parsing rules currently require that your keymap have nodes named `keymap` and `combos` that are nested one level-deep from the root.
  > (These conditions hold for most keymaps by convention.)

As an alternative to parsing, you can also check out the [examples](examples/) to find a layout similar to yours to use as a starting point.

### Tweaking the produced keymap representation

While the parsing step aims to create a decent starting point, you will likely want to make certain tweaks to the produced keymap representation.
Please refer to [the keymap schema specification](KEYMAP_SPEC.md) while making changes:

0. (If starting from a QMK keymap) Add combo definitions using key position indices.
1. Tweak the display form of parsed keys, e.g., replacing `&bootloader` with `BOOT`. (See [the customization section](#customization) to modify parser's behavior.)
2. If you have combos between non-adjacent keys or 3+ key positions, add `align` and/or `offset` properties in order to position them better
3. Add `type` specifiers to certain keys, such as `held` for layer keys used to enter the current layer or `ghost` for optional keys

It might be beneficial to start by `draw`'ing the current representation and iterate over these changes, especially for tweaking combo positioning.

### Producing the SVG

Final step is to produce the SVG representation using the **`keymap draw`** command.
However to do that, we need to specify the physical layout of the keyboard, i.e., how many keys there are, where each key is positioned etc.
You can provide this information to `keymap-drawer` in two ways:

- **QMK `info.json` specification**: Each keyboard in the QMK repo has a `info.json` file which specifies physical key locations.
  Using the keyboard name in the QMK repo, we can fetch this information from the [keyboard metadata API](https://docs.qmk.fm/#/configurator_architecture?id=keyboard-metadata):

  ```sh
  keymap draw -k ferris/sweep sweep_keymap.yaml >sweep_keymap.svg
  ```

  You can also specify a layout macro to use alongside the keyboard name if you don't want to use the default one:

  ```sh
  keymap draw -k crkbd/rev1 -l LAYOUT_split_3x5_3 corne_5col_keymap.yaml >corne_5col_keymap.svg
  ```

  `-j` flag also allows you to pass a local `info.json` file instead of the keyboard name.

  > **Note**
  >
  > If you parsed a QMK keymap, keyboard and layout information will be populated in the keymap YAML already, so you don't need to specify it in the command line.

  **Hint**: You can use the [QMK Configurator](https://config.qmk.fm/) to search for keyboard and layout names, and preview the physical layout.

- **Parametrized ortholinear layouts**: You can also specify parameters to automatically generate a split or non-split ortholinear layout, for example:

  ```sh
  keymap draw -o '{split: true, rows: 3, columns: 5, thumbs: 2}' sweep_keymap.yaml >sweep_keymap.ortho.svg
  ```

  See [the keymap specification](KEYMAP_SPEC.md#layout) for parameter definitions.

> **Note**
>
> If you prefer, you can specify physical layouts in the [keymap YAML file](KEYMAP_SPEC.md#layout) rather than the command line.
> This is also necessary for the web interface.

Once you produced the SVG representation, you can render it on your browser or use a tool like [CairoSVG](https://cairosvg.org/) or [Inkscape](https://inkscape.org/) to export to a different format.

## Customization

Both parsing and drawing can be customized using a configuration file passed to the `keymap` executable.
This allows you to, for instance, change the default keycode-to-symbol mappings while parsing, or change font sizes, colors etc. while drawing the SVG.

Start by dumping the default configuration settings to a file:

```sh
keymap dump-config >my_config.yaml
```

Then, edit the file to change the settings, referring to comments in [config.py](keymap_drawer/config.py).
You can then pass this file to either `draw` and `parse` subcommands with the `-c`/`--config` argument (note the location before the subcommand):

```sh
keymap -c my_config.yaml parse [...] >my_keymap.yaml
keymap -c my_config.yaml draw [...] my_keymap.yaml >my_keymap.svg
```

Since configuration classes are [Pydantic settings](https://docs.pydantic.dev/usage/settings/) they can also be overridden by environment variables with a `KEYMAP_` prefix:

```sh
KEYMAP_raw_binding_map='{"&bootloader": "BOOT"}' keymap parse -z zmk-config/config/cradio.keymap >cradio.yaml
```

Drawing parameters that are specified in the `draw_config` field can also be overridden in [the keymap YAML](KEYMAP_SPEC.md#draw_config).

## Development

This project requires Python 3.10+ and uses [Poetry](https://python-poetry.org/) for packaging.

To get started, [install Poetry](https://python-poetry.org/docs/#installation), clone this repo, then install dependencies with the `poetry` command:

```sh
git clone https://github.com/caksoylar/keymap-drawer.git
cd keymap-drawer
poetry install  # --with dev,lsp,streamlit optional dependencies
```

`poetry shell` will activate a virtual environment with the `keymap_drawer` module in Python path and `keymap` executable available.
Changes you make in the source code will be reflected when using the module or the command.

If you prefer not to use Poetry, You can get an editable install with `pip install --editable .` inside the `keymap-drawer` folder.

## Related projects

- [The original `keymap`](https://github.com/callum-oakley/keymap/)
- [Keymapviz](https://github.com/yskoht/keymapviz)
- [@nickcoutsos's ZMK keymap editor](https://github.com/nickcoutsos/keymap-editor)
- [@leiserfg's ZMK parser](https://github.com/leiserfg/zmk-config/tree/master/parser)
- [@jbarr21's ZMK parser](https://github.com/jbarr21/zmk-config/tree/main/parser)
