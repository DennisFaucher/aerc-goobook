# aerc-goobook

Use your **Google Contacts** as the address book for the [aerc](https://aerc-mail.org/) email client.

aerc can run an external command to tab-complete e-mail addresses
(`address-book-cmd`). This repository provides a small wrapper around
[goobook](https://pypi.org/project/goobook/) that adapts goobook's output to
the exact format aerc expects, so your Google Contacts show up in the `To:`,
`Cc:` and `Bcc:` fields.

## Features

- Tab-completion of Google Contacts inside aerc.
- Reads contacts from goobook's local cache, so lookups are fast and work
  offline between cache refreshes.
- Correctly handles goobook's leading blank line, its extra `kind`/`groups`
  column, and contact-group lines (which pack several addresses into one line).
- Escapes regex metacharacters in your query, so typing e.g. `(` does not crash
  goobook.
- No Python or other runtime dependencies of its own -- plain `bash`, `sed`
  and `awk`.

## How it works

aerc replaces `%s` in `address-book-cmd` with whatever you have typed after the
last comma, then runs the command with `sh -c`. It expects one completion per
line, tab-delimited, with the **e-mail address as the first field** and the
**contact name (optional) as the second field**.

goobook's `query` command does not quite match that format, so `aerc-goobook`:

1. runs `goobook query <your text>`,
2. drops goobook's leading blank line,
3. keeps only the e-mail and name columns,
4. splits contact-group lines into individual `email<TAB>name` entries.

## Requirements

- [aerc](https://aerc-mail.org/)
- [goobook](https://pypi.org/project/goobook/) (a Google Contacts client)
- A Google account
- `bash`, `sed`, `awk` (present on essentially every Unix-like system)

## Installation

### 1. Install goobook with pipx

[pipx](https://pipx.pypa.io/stable/) installs Python applications in isolated
environments and puts them on your `PATH`.

```sh
# Install pipx if you don't have it (Debian/Ubuntu example)
sudo apt install pipx
pipx ensurepath

# Install goobook
pipx install goobook
```

On other distributions, see the [pipx installation docs](https://pipx.pypa.io/stable/installation/).
After `pipx ensurepath`, restart your shell (or run `exec $SHELL -l`) so
`goobook` is found.

Verify it is available:

```sh
goobook --version
```

### 2. Create Google OAuth credentials for goobook

goobook talks to the [Google People API](https://developers.google.com/people),
which requires OAuth 2.0 credentials. You need a `client_secrets.json` file
(an OAuth **Desktop app** client).

Follow Google's official quickstart -- it walks through enabling the People API
and downloading the JSON file:

- **Google People API Python quickstart**: <https://developers.google.com/people/quickstart/python>

Or do it manually:

1. Create/select a project in the [Google Cloud Console](https://console.cloud.google.com/).
2. Enable the People API: <https://console.cloud.google.com/apis/library/people.googleapis.com>
3. Configure the OAuth consent screen (User type **External**):
   <https://console.cloud.google.com/apis/credentials/consent>
   - Add your own Google account under **Test users** (see troubleshooting below).
4. Create an OAuth client ID of type **Desktop app**:
   <https://console.cloud.google.com/apis/credentials>
5. **Download JSON** for that client and save it as `client_secrets.json`.

More background from Google:

- Create access credentials: <https://developers.google.com/workspace/guides/create-credentials>
- OAuth consent screen / publishing status: <https://support.google.com/cloud/answer/10311615>

### 3. Authenticate goobook

```sh
goobook authenticate -- client_secrets.json
```

A browser window opens; sign in with the Google account that owns the contacts
and grant access. goobook stores the token (and later a cache) under your XDG
data/config directories.

Test it:

```sh
goobook query alice
```

You should see matching contacts, one per line.

### 4. Install the aerc-goobook script

Clone this repository and make the script executable:

```sh
git clone https://github.com/DennisFaucher/aerc-goobook.git
chmod +x aerc-goobook/aerc-goobook
```

Optionally symlink it somewhere on your `PATH`:

```sh
mkdir -p ~/.local/bin
ln -s "$PWD/aerc-goobook/aerc-goobook" ~/.local/bin/aerc-goobook
```

### 5. Configure aerc

Add the following to the `[compose]` section of `~/.config/aerc/aerc.conf`
(use the absolute path to the script):

```ini
address-book-cmd=/path/to/aerc-goobook/aerc-goobook "%s"
```

The quotes around `%s` prevent anything you type from being interpreted by the
shell. You can also set this per account in `~/.config/aerc/accounts.conf`.

Restart aerc (or run `:reload`), then start typing an address in the `To:`,
`Cc:` or `Bcc:` field and press `<Tab>`.

## Usage

```sh
aerc-goobook <query>    # print "email<TAB>name" lines (this is what aerc calls)
aerc-goobook --reload   # force a refresh of the local goobook cache
aerc-goobook --help
```

The `GOOBOOK` environment variable can override the goobook binary path:

```sh
GOOBOOK=/usr/local/bin/goobook aerc-goobook alice
```

## Troubleshooting

**"Access blocked: ... has not completed the Google verification process"**

Your OAuth client is in *Testing* mode and the Google account you signed in
with is not listed as a test user. In the [Google Cloud Console](https://console.cloud.google.com/apis/credentials/consent),
add your address under **Test users**, then run `goobook authenticate` again.

**Re-authentication required every 7 days**

Apps in *Testing* status issue refresh tokens that expire after 7 days. To
avoid this, set the OAuth consent screen's **Publishing status** to
**In production** (you will see an "unverified app" warning you can dismiss via
*Advanced -> Go to app (unsafe)*, which is fine for personal use).

**No completions appear in aerc**

- Run `goobook query test` directly to confirm authentication works.
- Run `/path/to/aerc-goobook alice` directly; it should print `email<TAB>name`.
- Make sure the path in `address-book-cmd` is absolute and correct.

## License

Released under the MIT License.
