# Mage-OS Installer

`mageos-install` is a small bash tool that installs Mage-OS packages in your project through Composer.

It works with two plain text files:

* `mageos-base.conf` lists the base packages, which are always installed.
* `mageos-additional.conf` lists every other `mage-os/` package available on [Packagist](https://packagist.org). You choose which ones to install by uncommenting them.

All selected packages are installed with a single `composer require`, so Composer resolves the dependencies of the whole set at once.

## Requirements

* bash 3.2 or later (the version shipped with macOS is fine)
* `curl` and `jq`, to download the list of additional packages
* `jq` and `composer`, to install the packages
* an internet connection to reach packagist.org

The installer checks these commands before running and tells you which ones are missing.

## Installation

Clone or download this repository into a folder of your choice, and make sure the script is executable:

```sh
git clone <repository-url> mageos-installer
chmod +x mageos-installer/bin/mageos-install
```

The folder must keep this structure, because the script looks for its configuration files in the parent folder of `bin/`:

```
mageos-installer/
├── bin/
│   └── mageos-install
├── mageos-base.conf
├── mageos-additional.conf    (created by the init command)
└── README.md
```

Optionally, make the command available everywhere by adding the `bin/` folder to your `PATH`, or by creating a symlink (symlinks are followed, so the configuration files are still found):

```sh
ln -s "$PWD/mageos-installer/bin/mageos-install" /usr/local/bin/mageos-install
```

The examples below assume `mageos-install` is on your `PATH`.
Otherwise, call it with its full path, for example `/path/to/mageos-installer/bin/mageos-install install`.

## Quick start

1. Download the list of additional packages:

   ```sh
   mageos-install init
   ```

2. Open `mageos-additional.conf` and uncomment the packages you want (remove the `#` in front of the package name).

3. Go to the root of your project (the folder containing its `composer.json`) and install:

   ```sh
   cd /path/to/your/project
   mageos-install install
   ```

## Usage

```
mageos-install [options] <command>
```

| Command | Description |
|---------|-------------|
| `init` | Downloads the list of additional packages into `mageos-additional.conf` |
| `install` | Installs the packages of `mageos-base.conf` plus the ones selected in `mageos-additional.conf` |

| Option | Description |
|--------|-------------|
| `-d`, `--dry-run` | Runs the command without applying any change |
| `-h`, `--help` | Displays the help message |

Running `mageos-install` without arguments shows the help. Options can be placed before or after the command.

### `init`

```sh
mageos-install init
```

The command asks for confirmation, then queries Packagist for all packages published under the `mage-os/` vendor.
Packages already listed in `mageos-base.conf` are left out. The others are sorted by name and written to `mageos-additional.conf`,
each one commented out and preceded by its description:

```
# Adds a foo feature to the storefront
#mage-os/module-foo

# Integrates the bar service
#mage-os/module-bar
```

If `mageos-additional.conf` already exists, you are asked to confirm before it is overwritten.
**Overwriting discards your current selection**, so take note of the packages you uncommented if you want to select them again.

The file is written only when the download succeeds completely: if Packagist cannot be reached or returns an unexpected response,
the existing file is left untouched.

The `init` command does not depend on your project, so you can run it from any folder.

### Selecting packages

Edit `mageos-additional.conf` and uncomment the packages you want to install:

```
# Adds a foo feature to the storefront
mage-os/module-foo

# Integrates the bar service
#mage-os/module-bar
```

Here `mage-os/module-foo` is selected and `mage-os/module-bar` is not.

You can also pin a version by adding a Composer constraint after a colon:

```
mage-os/module-foo:^1.2
```

Without a constraint, Composer picks the latest version compatible with your project.

### `install`

Run the command from the root of your project, because Composer is executed in the current folder:

```sh
cd /path/to/your/project
mageos-install install
```

The command:

1. collects the active packages of `mageos-base.conf` and `mageos-additional.conf`;
2. reports the packages that your `composer.json` already requires (they are still passed to Composer);
3. shows the full list and asks for confirmation, for example `12 packages will be installed through Composer. Proceed? [y/N]`;
4. runs a single `composer require` with all the packages.

The installer does not run anything after Composer. If your project needs further steps (for example `bin/magento setup:upgrade`), run them yourself.

### Dry run

Use `--dry-run` to see what would happen without changing anything:

```sh
mageos-install --dry-run init      # prints the content of mageos-additional.conf without writing it
mageos-install --dry-run install   # runs "composer require --dry-run" with the selected packages
```

Prompts and messages are written to standard error, so you can save a preview of the file:

```sh
mageos-install --dry-run init > preview.conf
```

## Configuration files

### `mageos-base.conf`

The curated list of base packages, installed every time you run `install`.
It ships with the installer and the script never modifies it. If it is missing, both commands stop with an error:
restore it from this repository.

### `mageos-additional.conf`

Generated by `init`. Edit it freely to select packages, but keep in mind that running `init` again overwrites it.

### Line format

Both files use the same format. Leading and trailing spaces are ignored.

| Line | Meaning |
|------|---------|
| `mage-os/module-foo` | Active package: it will be installed |
| `mage-os/module-foo:^1.2` | Active package with a version constraint |
| `#mage-os/module-foo` or `# mage-os/module-foo` | Commented package: ignored |
| `# Any other text` | Description: ignored |
| empty line | Ignored |

Any other line (for example `mage-os/module-foo ^1.2`, with a space instead of a colon, or a name with uppercase letters)
is ignored and reported with a warning showing the file name and line number, so a typo never goes unnoticed.

If the same package is listed more than once with the same constraint, it is installed once.
If it is listed with different constraints, `install` stops with an error showing where the conflicting lines are.

## Confirmation prompts

Every question shows `[y/N]`. The default answer is **no**.

* `y`, `Y`, `yes`, `Yes`, `YES` mean yes.
* `n`, `N`, `no`, `No`, `NO`, or just Enter mean no.
* Any other answer makes the question appear again.

Answering no stops the installer with the message "Ended by the user".

## Running inside a container

If Composer runs inside a container (Docker, Warden, DDEV and similar), run `install` inside the container, from the project root,
with the installer folder reachable from there. `curl`, `jq` and `composer` must be available in the container.

Since `init` does not need your project, you can also run it on your machine and only run `install` inside the container,
as long as both see the same installer folder.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success, help displayed, or ended by the user |
| 1 | Error: missing file or command, network failure, invalid Packagist response, no package found, file write failure, conflicting constraints, nothing to install, unknown or missing command, unknown option |
| other | The installation failed in Composer: the installer exits with the Composer exit code |

## Troubleshooting

**`missing required command(s): ...`**
Install the listed commands with your package manager (for example `brew install jq` or `apt install jq curl`).

**`mageos-additional.conf not found`**
Run `mageos-install init` first.

**`base configuration file not found`**
`mageos-base.conf` must be in the parent folder of `bin/`. Restore it from this repository.

**`no packages to install`**
`mageos-base.conf` has no active packages and nothing is uncommented in `mageos-additional.conf`.

**Composer fails**
Composer's own output explains the reason (usually a version conflict). Adjust the constraints or the selection,
and use `mageos-install --dry-run install` to check the result before installing.
