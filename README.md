# elixir-pdf-generator

A wrapper for both wkhtmltopdf and chrome-headless plus PDFTK (adds in
encryption) for use in Elixir projects. 

If available, it will use xvfb-run (x
virtual frame buffer) to use wkhtmltopdf on systems that have no X installed,
e.g. a server.

# New in 0.5.0 - farewell Porcelain, hello chrome-headless (puppeteer)

  - 0.5.0
    - **Got rid of Porcelain** dependency as it interferes with many builds using
      plain `System.cmd/3`. Please note, that as of the documentation
      (https://hexdocs.pm/elixir/System.html#cmd/3) ports will be closed but in
      case wkhtmltopdf somehow hangs, nobody takes care of terminating it.
    - Refactored some sections
    - **Support URLs** instead of just plain HTML
    - **Support for chrome-headless** for (at least for me) faster and nicer renderings.
    - Since this is hopefully helpful, I rose the version to 0.5.0 even tough
      the API stays consistent
  - 0.5.1
    - allow chrome to be executed as root via default config option
      `disable_chrome_sandbox` – this is required for an easy usage within a
      docker container as in
      [elixir-pdf-server](https://github.com/gutschilla/elixir-pdf-server)
  - 0.5.2
    - **BUGFIX** introduced in 0.5.0 when global options to wkhtmltopdf weren't
      accepted any more due to wrong shell parameter order. Thanks to
      [manukall](https://github.com/manukall) for reporting.
  - 0.5.3
    - **BUGFIX** introduced in 0.5.0 when certain shells don't accept
      `["foo=bar", …]` parameters which should correctly be `["foo", "bar"]`
      Thanks to [@egze](https://github.com/egze) for submitting a patch.
    - Refactored `PathAgent` that holds configuration state for readability and
      more fashionable and extensible error messages. Extensible towards new
      generators.
    - Updated README to be more elaborative on how to install `wkhtmltopdf` and
      `chrome-headless-render-pdf`

For a proper changelog, see [CHANGES](CHANGES.md)

# System prerequisites 

It's either 

* wkhtmltopdf or 

* nodejs and possibly chrome/chromium

## chrome-headless

This will allow you to make more use of Javascript and advanced CSS as it's just
your Chrome/Chromium browser rendering your web page as HTML and printing it as
PDF. Rendering _tend_ to be a bit faster than with wkhtmltopdf. The price tag is
that PDFs printed with chrome/chromium are usually considerably bigger than
those generated with wkhtmltopdf.

1. Run `npm -g install chrome-headless-render-pdf puppeteer`. 

   This requires [nodejs](https://nodejs.org), of course. This will install a
   recent chromium and chromedriver to run Chrome in headless mode and use this
   browser and its API to print PDFs globally on your machine.
   
   If you prefer a project-local install, just use `npm install` This will
   install dependencies under `./node_modules`. Be aware that those won't be
   packaged in your distribution (I will add support for this later).

   On some machines, this doesn't install Chromium and fails. Here's how to get
   this running on Ubtunu 18:
   
   ```
   DEBIAN_FRONTEND=noninteractive PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=TRUE \
     apt-get install -y chromium-chromedriver \
     && npm -g install chrome-headless-render-pdf puppeteer
   ```
   
## wkhtmltopdf

2. Download wkhtmltopdf and place it in your $PATH. Current binaries can be
   found here: http://wkhtmltopdf.org/downloads.html
   
   For the impatient:
   
   ```
   apt-get -y install xfonts-base xfonts-75dpi \
    && wget https://downloads.wkhtmltopdf.org/0.12/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb \
    && dpkg -i wkhtmltox_0.12.5-1.bionic_amd64.deb
   ```

3. _optional:_ Install `xvfb` (shouldn't be required with the binary mentioned above):

   To use other wkhtmltopdf executables comiled with an unpatched Qt on systems
   without an X window server installed, please install `xvfb-run` from your
   repository (on Debian/Ubuntu: `sudo apt-get install xvfb`).
   
   I haven't heard any feedback of people using this feature since a while since
   the wkhtmltopdf projects ships ready-made binaries. I will deprecate this
   starting in `0.6.0` since, well, YAGNI.

4. _optional:_ Install `pdftk` via your package manager or homebrew. The project
   page also contains a Windows installer. On Debian/Ubuntu just type:
   `apt-get -y install pdftk`

# Usage

Add this to your dependencies in your mix.exs:

```Elixir
    def application do
        [applications: [
            :logger,
            :pdf_generator # <-- add this
        ]]
    end

    defp deps do
        [
            # ... whatever else
            { :pdf_generator, ">=0.5.3" }, # <-- and this
        ]
    end
```

Then pass some html to PdfGenerator.generate

```Elixir
$ iex -S mix

html = "<html><body><p>Hi there!</p></body></html>"
# be aware, this may take a while...
{:ok, filename}    = PdfGenerator.generate(html, page_size: "A5")
{:ok, pdf_content} = File.read(filename)

# or, if you prefer methods that raise on error:
filename = PdfGenerator.generate!(html, generator: :chrome)
```

Or, pass some URL

```
PdfGenerator.generate {:url, "http://google.com"}, page_size: "A5"
```

Or, use chrome-headless

```
html_works_too = "<html><body><h1>Minimalism!"
{:ok, filename}    = PdfGenerator.generate html_works_too, generator: :chrome
```

Or use the bang-methods:

```Elixir
filename   = PdfGenerator.generate! "<html>..."
pdf_binary = PdfGenerator.generate_binary! "<html>..."
```

# Options and Configuration

This module will automatically try to finde both `wkhtmltopdf` and `pdftk` in
your path. But you may override or explicitly set their paths in your
`config/config.exs`.

```Elixir
config :pdf_generator,
    wkhtml_path:    "/usr/bin/wkhtmltopdf",   # <-- this program actually does the heavy lifting
    pdftk_path:     "/usr/bin/pdftk"          # <-- only needed for PDF encryption
```

or, if you prefer shrome-headless

```
config :pdf_generator,
    use_chrome: true             # <-- will be default by 0.6.0
    pdftk_path: "/usr/bin/pdftk" # <-- only needed for PDF encryption
```

## Running wkhtml headless (server-mode)

This section only applies to `wkhtmltopdf` users.

If you want to run `wkhtmltopdf` with an unpatched verison of webkit that requires
an X Window server, but your server (or Mac) does not have one installed,
you may find the `command_prefix` handy:

```Elixir
PdfGenerator.generate "<html..", command_prefix: "xvfb-run"
```

This can also be configured globally in your `config/config.exs`:

```Elixir
config :pdf_generator,
    command_prefix: "/usr/bin/xvfb-run"
```

If you will be generating multiple PDFs simultaneously, or in rapid succession,
you will need to configure `xvfb-run` to search for a free X server number,
or set the server number explicitly. You can use the `command_prefix` to pass
options to the `xvfb-run` command.

```Elixir
config :pdf_generator,
    command_prefix: ["xvfb-run", "-a"]
```

## More options

- `filename` - filename for the output pdf file (without .pdf extension, defaults to a random string)

- `page_size`:
  * defaults to `A4`, see `wkhtmltopdf` for more options
  * A4 will be translated to `page-height 11` and `page-width 8.5` when
    chrome-headless is used

- `open_password`:    requires `pdftk`, set password to encrypt PDFs with

- `edit_password`:    requires `pdftk`, set password for edit permissions on PDF

- `shell_params`:     pass custom parameters to `wkhtmltopdf`. **CAUTION: BEWARE OF SHELL INJECTIONS!**

- `command_prefix`:   prefix `wkhtmltopdf` with some command or a command with options
                      (e.g. `xvfb-run -a`, `sudo` ..)

- `delete_temporary`: immediately remove temp files after generation

## Heroku Setup

If you want to use this project on heroku, you can use buildpacks instead of binaries
to load `pdftk` and `wkhtmltopdf`:
```
https://github.com/fxtentacle/heroku-pdftk-buildpack
https://github.com/dscout/wkhtmltopdf-buildpack
https://github.com/HashNuke/heroku-buildpack-elixir
https://github.com/gjaldon/phoenix-static-buildpack
```

__note:__ The list also includes Elixir and Phoenix buildpacks to show you that they
must be placed after `pdftk` and `wkhtmltopdf`. It won't work if you load the
Elixir and Phoenix buildpacks first.

# Documentation

For more info, read the [docs on hex](http://hexdocs.pm/pdf_generator) or issue
`h PdfGenerator` in your iex shell.
