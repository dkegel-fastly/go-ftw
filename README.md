# Go-FTW - Framework for Testing WAFs in Go!

[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)](https://github.com/pre-commit/pre-commit)
[![Go Report Card](https://goreportcard.com/badge/github.com/fzipi/go-ftw)](https://goreportcard.com/report/github.com/fzipi/go-ftw)
[![Go Doc](https://img.shields.io/badge/godoc-reference-blue.svg?style=flat-square)](http://godoc.org/github.com/fzipi/go-ftw)
[![PkgGoDev](https://pkg.go.dev/badge/github.com/fzipi/go-ftw)](https://pkg.go.dev/github.com/fzipi/go-ftw)
[![Release](https://img.shields.io/github/v/release/fzipi/go-ftw.svg?style=flat-square)](https://github.com/fzipi/go-ftw/releases/latest)
[![Total alerts](https://img.shields.io/lgtm/alerts/g/fzipi/go-ftw.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/fzipi/go-ftw/alerts/)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=fzipi_go-ftw&metric=coverage)](https://sonarcloud.io/dashboard?id=fzipi_go-ftw)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=fzipi_go-ftw&metric=alert_status)](https://sonarcloud.io/dashboard?id=fzipi_go-ftw)


This software should be compatible with the [Python version](https://pypi.org/project/ftw/).

I wrote this one to get more insights on the original version, and trying to shed some light on the internals. There are many assumptions on the inner workings that I needed to dig into the code to know how they worked.

My goals are:
- get a compatible `ftw` version, with no dependencies and easy to deploy
- be extremely CI/CD friendly
- be fast (if possible)
- add features like:
  - syntax checking on the test files
  - use docker API to get logs (if possible), so there is no need to read files
  - add different outputs for CI (junit xml?, github, gitlab, etc.)

## Install

Go to the [releases](https://github.com/fzipi/go-ftw/releases) page and get the one that matches your OS.

If you have Go installed and configured to run Go binaries from your shell you can also run
```bash
go install github.com/fzipi/go-ftw@latest
```

## Example Usage

To run tests you need:
1. a WAF (doh!)
2. a file where the waf stores the logs
3. a config file, or environment variables, with the information to get the logs and how to parse them (I might embed this for the most commonly used, like Apache/NGiNX)

By default, _ftw_ would search for a file in `$PWD` with the name `.ftw.yaml`. Example configurations for `apache` and `nginx` below:

```yaml
---
logfile: '../coreruleset/tests/logs/modsec2-apache/apache2/error.log'
```

```yaml
---
logfile: '../coreruleset/tests/logs/modsec3-nginx/nginx/error.log'
```

I normally perform my testing using the [Core Rule Set](https://github.com/coreruleset/coreruleset/).

You can start the containers from that repo using docker-compose:

```bash
git clone https://github.com/coreruleset/coreruleset.git
docker-compose -f tests/docker-compose.yml up -d modsec2-apache
```

This is the help for the `run` command:
```bash
❯ ftw run -h
Run all tests below a certain subdirectory. The command will search all y[a]ml files recursively and pass it to the test engine.

Usage:
  ftw run [flags]

Flags:
  -d, --dir string       recursively find yaml tests in this directory (default ".")
  -e, --exclude string   exclude tests matching this Go regexp (e.g. to exclude all tests beginning with "91", use "91.*").
                         If you want more permanent exclusion, check the 'testmodify' option in the config file.
  -h, --help             help for run
      --id string        (deprecated). Use --include matching your test only.
  -i, --include string   include only tests matching this Go regexp (e.g. to include only tests beginning with "91", use "91.*").
  -q, --quiet            do not show test by test, only results
  -t, --time             show time spent per test

Global Flags:
      --config string   override config file (default is $PWD/.ftw.yaml) (default "c")
      --debug           debug output
      --trace           trace output: really, really verbose

```

Here's an example on how to run your tests:

```bash
ftw run -d tests -t
```

And the result should be similar to:

```bash
❯ ./ftw run -d tests -t

🛠️  Starting tests!
🚀 Running!
👉 executing tests in file 911100.yaml
	running 911100-1: ✔ passed 6.382692ms
	running 911100-2: ✔ passed 4.590739ms
	running 911100-3: ✔ passed 4.833236ms
	running 911100-4: ✔ passed 4.675082ms
	running 911100-5: ✔ passed 3.581742ms
	running 911100-6: ✔ passed 6.426949ms
...
	running 944300-322: ✔ passed 13.292549ms
	running 944300-323: ✔ passed 8.960695ms
	running 944300-324: ✔ passed 7.558008ms
	running 944300-325: ✔ passed 5.977716ms
	running 944300-326: ✔ passed 5.457394ms
	running 944300-327: ✔ passed 5.896309ms
	running 944300-328: ✔ passed 5.873305ms
	running 944300-329: ✔ passed 5.828122ms
➕ run 2354 total tests in 18.923445528s
⏭ skipped 7 tests
🎉 All tests successful!
```
Happy testing!

## Additional features

You can add functions to your tests, to simplify bulk writing, or even read values from the environment while executing. This is because `data:` sections in tests are parsed with Go [text/template](https://golang.org/pkg/text/template/), and also are given the power of additional [Sprig functions](https://masterminds.github.io/sprig/).

This will allow you to write tests like this:

```yaml
data: 'foo=%3d{{ "+" | repeat 34 }}'
```

Will be expanded to:

```yaml
data: 'foo=%3d++++++++++++++++++++++++++++++++++'
```

But also, you can get values from the environment dynamically when the test is run:

```yaml
data: 'username={{ env "USERNAME" }}
```

Will give you, as you expect, the username running the tests

```yaml
data: 'username=fzipi
```

Other interesting functions you can use are: `randBytes`, `htpasswd`, `encryptAES`, etc.

## Overriding test results

Sometimes you have tests that work well for some platform combinations, e.g. Apache + modsecurity2, but fail for others, e.g. NGiNX + modsecurity3. Taking that into account, you can override test results using the `testoverride` config param. The test will be run, but the _result_ would be overriden, and your comment will be printed out.

Example:

```yaml
...
testoverride:
  ignore:
    # text comes from our friends at https://github.com/digitalwave/ftwrunner
    '941190-3': 'known MSC bug - PR #2023 (Cookie without value)'
    '941330-1': 'know MSC bug - #2148 (double escape)'
    '942480-2': 'known MSC bug - PR #2023 (Cookie without value)'
    '944100-11': 'known MSC bug - PR #2045, ISSUE #2146'
  forcefail:
    '123456-01': 'I want this test to fail, even if passing'
  forcepass:
    '123456-02': 'This test will always pass'
```

You can combine any of `ignore`, `forcefail` and `forcepass` to make it work for you.

## How log parsing works
The log output from your WAF is parsed and compared to the expected output.
The problem with log files is that they aren't updated in real time, e.g. because the
web server / WAF has an internal buffer, or because there's some `fsync` magic involved).
To make log parsing consistent and guarantee that we will see output when we need it,
go-ftw uses "log markers". In essence, unique log entries are written _before_ and _after_
every test stage. go-ftw can then search for these markers.

The [container images for Core Rule Set](https://github.com/coreruleset/modsecurity-crs-docker) can be configured to write these marker log lines by setting
the `CRS_ENABLE_TEST_MARKER` environment variable. If you are testing a different WAF
you will need to instrument it with the same idea (unless you are using "cloud mode").
The rule for CRS looks like this:
```
# Write the value from the X-CRS-Test header as a marker to the log
SecRule REQUEST_HEADERS:X-CRS-Test "@rx ^.*$" \
  "id:999999,\
  phase:1,\
  log,\
  msg:'%{MATCHED_VAR}',\
  pass,\
  t:none"
```

The rule looks for an HTTP header named `X-CRS-Test` and writes its value to the log,
the value being the UUID of a test stage.

You can configure the name of the HTTP header by setting the `logmarkerheadername`
option in the configuration to a custom value (the value is case insensitive).

## License
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Ffzipi%2Fgo-ftw.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Ffzipi%2Fgo-ftw?ref=badge_large)
