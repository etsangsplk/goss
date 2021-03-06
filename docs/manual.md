# goss manual

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Usage](#usage)
  * [global options](#global-options)
    * [\-g gossfile](#-g-gossfile)
  * [commands](#commands)
    * [add, a \- Add system resource to test suite](#add-a---add-system-resource-to-test-suite)
    * [autoadd, aa \- Auto add all matching resources to test suite](#autoadd-aa---auto-add-all-matching-resources-to-test-suite)
    * [render, r \- Render gossfile after importing all referenced gossfiles](#render-r---render-gossfile-after-importing-all-referenced-gossfiles)
    * [serve, s \- Serve a health endpoint](#serve-s---serve-a-health-endpoint)
    * [validate, v \- Validate the system](#validate-v---validate-the-system)
* [Important note about goss file format](#important-note-about-goss-file-format)
* [Available tests](#available-tests)
  * [addr](#addr)
  * [command](#command)
  * [dns](#dns)
  * [file](#file)
  * [gossfile](#gossfile)
  * [group](#group)
  * [http](#http)
  * [interface](#interface)
  * [kernel-param](#kernel-param)
  * [mount](#mount)
  * [package](#package)
  * [port](#port)
  * [process](#process)
  * [service](#service)
  * [user](#user)
* [Patterns](#patterns)
* [Advanced Matchers](#advanced-matchers)
* [Templates](#templates)

## Usage

```
NAME:
   goss - Quick and Easy server validation

USAGE:
   goss [global options] command [command options] [arguments...]

VERSION:
   0.0.0

COMMANDS:
   validate, v	Validate system
   serve, s	Serve a health endpoint
   render, r	render gossfile after imports
   autoadd, aa	automatically add all matching resource to the test suite
   add, a	add a resource to the test suite
   help, h	Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --gossfile, -g "./goss.yaml"	Goss file to read from / write to [$GOSS_FILE]
   --vars value                 json/yaml file containing variables for template [$GOSS_VARS]
   --package 			Package type to use [rpm, deb, apk, pacman]
   --help, -h			show help
   --generate-bash-completion
   --version, -v		print the version

```
**Note:** *Most flags can be set by using environment variables, see `--help` for more info.*


## global options
### -g gossfile
The file to use when reading/writing tests. Use `-g -` to read from `STDIN`.

Valid formats:
* **YAML** (default)
* **JSON**

### --vars
The file to read variables from when rendering gossfile [templates](#templates).

Valid formats:
* **YAML** (default)
* **JSON**

### --package <type>
The package type to check for.

Valid options are:
* `apk`
* `deb`
* `pacman`
* `rpm`


## commands
Commands are the actions goss can run.

* [add](#add-a---add-system-resource-to-test-suite): add a single test for a resource
* [autoadd](#autoadd-aa---auto-add-all-matching-resources-to-test-suite): automatically add multiple tests for a resource
* [render](#render-r---render-gossfile-after-importing-all-referenced-gossfiles): renders and outputs the gossfile, importing all included gossfiles
* [serve](#serve-s---serve-a-health-endpoint): serves the gossfile validation as an HTTP endpoint on a specified address and port, so you can use your gossfile as a health repor for the host
* [validate](#validate-v---validate-the-system): runs the goss test suite on your server


### add, a - Add system resource to test suite

This will add a test for a resource. Non existent resources will add a test to ensure they do not exist on the system. A sub-command *resource type* has to be provided when running `add`.

#### Resource types
* `addr` - can verify if a remote `address:port` is reachable, see [addr](#addr)
* `command` - can run a [command](#command) and validate the exit status and/or output
* `dns` - resolves a [dns](#dns) name and validates the addresses
* `file` - can validate a [file](#file) existence, permissions, stats (size, etc) and contents
* `goss` - allows you to include the contents of another [gossfile](#gossfile)
* `group` - can validate the existence and values of a [group](#group) on the system
* `http` - can validate the HTTP response code and content of a URI, see [http](#http)
* `interface` - can validate the existence and values (es. the addresses) of a network interface, see [interface](#interface)
* `kernel-param` - can validate kernel parameters (sysctl values), see [kernel-param](#kernel-param)
* `mount` - can validate the existence and options relative to a [mount](#mount) point
* `package` - can validate the status of a [package](#package) using the package manager specified on the commandline with `--package`
* `port` - can validate the status of a local [port](#port), for example `80` or `udp:123`
* `process` - can validate the status of a [process](#process)
* `service` - can validate if a [service](#service) is running and/or enabled at boot
* `user` - can validate the existence and values of a [user](#user) on the system

#### Flags
##### --exclude-attr
Ignore **non-required** attribute(s) matching the provided glob when adding a new resource, may be specified multiple times.

#### Example:
```bash
$ goss a file /etc/passwd
$ goss a user nobody
$ goss a --exclude-attr home --exclude-attr shell user nobody
$ goss a --exclude-attr '*' user nobody
```


### autoadd, aa - Auto add all matching resources to test suite
Automatically [adds](#add-a---add-system-resource-to-test-suite) all **existing** resources matching the provided argument.

Will automatically add the following matching resources:
* `file` - only if argument contains `/`
* `group`
* `package`
* `port`
* `process` - Also adding any ports it's listening to (if run as root)
* `service`
* `user`

Will **NOT** automatically add:
* `addr`
* `command` - for safety
* `dns`
* `http`
* `interface`
* `kernel-param`
* `mount`

#### Example:
```bash
$ goss autoadd sshd
```

Generates the following `goss.yaml`
```yaml
port:
  tcp:22:
    listening: true
    ip:
    - 0.0.0.0
  tcp6:22:
    listening: true
    ip:
    - '::'
service:
  sshd:
    enabled: true
    running: true
user:
  sshd:
    exists: true
    uid: 74
    gid: 74
    groups:
    - sshd
    home: /var/empty/sshd
    shell: /sbin/nologin
group:
  sshd:
    exists: true
    gid: 74
process:
  sshd:
    running: true
```


### render, r - Render gossfile after importing all referenced gossfiles
This command allows you to keep your tests separated and render a single, valid, gossfile, by including them with the `gossfile` directive.

#### Flags
##### --debug
This prints the rendered golang template prior to printing the parsed JSON/YAML gossfile.

#### Example:
```bash

$ cat goss_httpd_package.yaml
package:
  httpd:
    installed: true
    versions:
    - 2.2.15

$ cat goss_httpd_service.yaml
service:
  httpd:
    enabled: true
    running: true

$ cat goss_nginx_service-NO.yaml
service:
  nginx:
    enabled: false
    running: false

$ cat goss.yaml
gossfile:
  goss_httpd_package.yaml: {}
  goss_httpd_service.yaml: {}
  goss_nginx_service-NO.yaml: {}

$ goss -g goss.yaml render
package:
  httpd:
    installed: true
    versions:
    - 2.2.15
service:
  httpd:
    enabled: true
    running: true
  nginx:
    enabled: false
    running: false
```


### serve, s - Serve a health endpoint

`serve` exposes the goss test suite as a health endpoint on your server. The end-point will return the stest results in the format requested and an http status of 200 or 503.

`serve` will look for a test suite in the same order as [validate](#validate-v---validate-the-system)

#### Flags
* `--cache <value>`, `-c <value>` - Time to cache the results (default: 5s)
* `--endpoint <value>`, `-e <value>` - Endpoint to expose (default: `/healthz`)
* `--format`, `-f` - output format, same as [validate](#validate-v---validate-the-system)
* `--listen-addr [ip]:port`, `-l [ip]:port` - Address to listen on (default: `:8080`)
* `--max-concurrent` - Max number of tests to run concurrently

#### Example:

```bash
$ goss serve &
$ curl http://localhost:8080/healthz

# JSON endpoint
$ goss serve --format json &
$ curl localhost:8080/healthz
```


### validate, v - Validate the system

`validate` runs the goss test suite on your server. Prints an rspec-like (by default) output of test results. Exits with status 0 on success, non-0 otherwise.

#### Flags
* `--format`, `-f` (output format)
  * `documentation` - Verbose test results
  * `JSON` - Detailed test result
  * `JUnit`
  * `nagios` - Nagios/Sensu compatible output /w exit code 2 for failures.
  * `nagios_verbose` - Nagios output with verbose failure output.
  * `rspecish` **(default)** - Similar to rspec output
  * `TAP`
* `--max-concurrent` - Max number of tests to run concurrently
* `--no-color` - Disable color
* `--retry-timeout`, `-r` - Retry on failure so long as elapsed + sleep time is less than this (default: 0)
* `--sleep`, `-s` - Time to sleep between retries (default: 1s)

#### Example:

```bash
$ goss validate --format documentation
File: /etc/hosts: exists: matches expectation: [true]
DNS: localhost: resolveable: matches expectation: [true]
[...]
Total Duration: 0.002s
Count: 10, Failed: 2, Skipped: 0

$ curl -s https://static/or/dynamic/goss.json | goss validate
...F.F
[...]
Total Duration: 0.002s
Count: 6, Failed: 2, Skipped: 0

$ goss render | ssh remote-host 'goss -g - validate'
......

Total Duration: 0.002s
Count: 6, Failed: 0, Skipped: 0
```



## Important note about goss file format
It is important to note that both YAML and JSON are formats that describe a nested data structure.

**WRONG way to write a goss file**

```yaml
file:
  /etc/httpd/conf/httpd.conf:
    exists: true

service:
  httpd:
    enabled: true
    running: true

file:
  /var/www/html:
    filetype: directory
    exists: true  
```

If you try to validate this file, it will **only** run the second `file` test:

```bash
# goss validate --format documentation
File: /var/www/html: exists: matches expectation: [true]
File: /var/www/html: filetype: matches expectation: ["directory"]
Service: httpd: enabled: matches expectation: [true]
Service: httpd: running: matches expectation: [true]

Total Duration: 0.014s
Count: 8, Failed: 0, Skipped: 0
```

As you can see, the first `file` check has not been run because the second `file` entry *overwrites* the previous one.

You need to make sure all the entries of the same type are under the same declaration.

**This is the CORRECT way to write a goss file**

```yaml
file:
  /etc/httpd/conf/httpd.conf:
    exists: true
  /var/www/html:
    filetype: directory
    exists: true  

service:
  httpd:
    enabled: true
    running: true
```

Running validate with this configuration will correctly check both files:

```bash
# goss validate --format documentation
File: /var/www/html: exists: matches expectation: [true]
File: /var/www/html: filetype: matches expectation: ["directory"]
File: /etc/httpd/conf/httpd.conf: exists: matches expectation: [true]
Service: httpd: enabled: matches expectation: [true]
Service: httpd: running: matches expectation: [true]

Total Duration: 0.014s
Count: 10, Failed: 0, Skipped: 0
```

Please note that using the `goss add` and `goss autoadd` command will create a valid file, but if you're writing your files by hand you'll save a lot of time by taking this in consideration.

If you want to keep your tests in separate files, the best way to obtain a single, valid, file is to create a main goss file that includes the others with the [gossfile](#gossfile) directive and then [render](#render-r---render-gossfile-after-importing-all-referenced-gossfiles) it.



## Available tests

* [addr](#addr)
* [command](#command)
* [dns](#dns)
* [file](#file)
* [gossfile](#gossfile)
* [group](#group)
* [http](#http)
* [interface](#interface)
* [kernel-param](#kernel-param)
* [mount](#mount)
* [package](#package)
* [port](#port)
* [process](#process)
* [service](#service)
* [user](#user)


### addr
Validates if a remote `address:port` are accessible.

```yaml
tcp://ip-address-or-domain-name:80:
  reachable: true
  timeout: 500
```


### command
Validates the exit-status and output of a command

```yaml
command:
  go version:
    # required attributes
    exit-status: 0
    # optional attributes
    stdout:
    - go version go1.6 linux/amd64
    stderr: []
    timeout: 10000 # in milliseconds
```

`stdout` and `stderr` can be a string or [pattern](#patterns)


### dns
Validates that the provided address is resolveable and the addrs it resolves to.

```yaml
dns:
  localhost:
    # required attributes
    resolveable: true
    # optional attributes
    server: 8.8.8.8
    addrs:
    - 127.0.0.1
    - ::1
    timeout: 500 # in milliseconds
```

With the server attribute set, it is possible to validate the following types of DNS record:

- A
- AAAA
- CNAME
- MX
- NS
- PTR
- SRV
- TXT

To validate specific DNS address types, prepend the hostname with the type and a colon, a few examples:

```yaml
dns:
  # Validate a CNAME record
  CNAME:dnstest.github.io:
    resolveable: true
    server: 8.8.8.8
    addrs:
    - "github.map.fastly.net."

  # Validate a PTR record
  PTR:8.8.8.8:
    resolveable: true
    server: 8.8.8.8
    addrs:
    - "google-public-dns-a.google.com."

  # Validate and SRV record
  SRV:_https._tcp.dnstest.io:
    resolveable: true
    server: 8.8.8.8
    addrs:
    - "0 5 443 a.dnstest.io."
    - "10 10 443 b.dnstest.io."
```

Please note that if you want `localhost` to **only** resolve `127.0.0.1` you'll need to use [Advanced Matchers](#advanced-matchers)

```yaml
dns:
  localhost:
    resolveable: true
    addrs:
      consist-of: [127.0.0.1]
    timeout: 500 # in milliseconds
```

### file
Validates the state of a file

```yaml
file:
  /etc/passwd:
    # required attributes
    exists: true
    # optional attributes
    mode: "0644"
    size: 2118 # in bytes
    owner: root
    group: root
    filetype: file # file, symlink, directory
    contains: [] # Check file content for these patterns
    md5: 7c9bb14b3bf178e82c00c2a4398c93cd # md5 checksum of file
```

`contains` can be a string or a [pattern](#patterns)


### gossfile
Import other gossfiles from this one. This is the best way to maintain a large mumber of tests, and/or create profiles. See [render](#render-r---render-gossfile-after-importing-all-referenced-gossfiles) for more examples.

```yaml
gossfile:
  goss_httpd.yaml: {}
```


### group
Validates the state of a group

```yaml
group:
  nfsnobody:
    # required attributes
    exists: true
    # optional attributes
    gid: 65534
```


### http
Validates HTTP response status code and content.

```yaml
http:
  https://www.google.com:
    # required attributes
    status: 200
    # optional attributes
    allow-insecure: false
    no-follow-redirects: false # Setting this to true will NOT follow redirects
    timeout: 1000
    body: [] # Check http response content for these patterns
```


### interface
Validates network interface values

```yaml
interface:
  eth0:
    # required attributes
    exists: true
    # optional attributes
    addrs:
    - 172.17.0.2/16
    - fe80::42:acff:fe11:2/64
```


### kernel-param
Validates kernel param (sysctl) value.

```yaml
kernel-param:
  kernel.ostype:
    # required attributes
    value: Linux
```

To see the full list of current values, run `sysctl -a`.


### mount
Validates mount point attributes.

```yaml
mount:
  /home:
    # required attributes
    exists: true
    # optional attributes
    opts:
    - rw
    - relatime
    source: /dev/mapper/fedora-home
    filesystem: xfs
```


### package
Validates the state of a package

```yaml
package:
  httpd:
    # required attributes
    installed: true
    # optional attributes
    versions:
    - 2.2.15
```

**NOTE:** this check uses the `--package <format>` parameter passed on the command line.


### port
Validates the state of a local port.

```yaml
port:
  # {tcp,tcp6,udp,udp6}:port_num
  tcp:22:
    # required attributes
    listening: true
    # optional attributes
    ip: # what IP(s) is it listening on
    - 0.0.0.0
```


### process
Validates if a process is running.

```yaml
process:
  chrome:
    # required attributes
    running: true
```


### service
Validates the state of a service.

```yaml
service:
  sshd:
    # required attributes
    enabled: true
    running: true
```

**NOTE:** this will **not** automatically check if the process is alive, it will check the status from `systemd`/`upstart`/`init`.


### user
Validates the state of a user

```yaml
user:
  nfsnobody:
    # required attributes
    exists: true
    # optional attributes
    uid: 65534
    gid: 65534
    groups:
    - nfsnobody
    home: /var/lib/nfs
    shell: /sbin/nologin
```



## Patterns
For the attributes that use patterns (ex. `file`, `command` `output`), each pattern is checked against the attribute string, the type of patterns are:

* `"string"` - checks if any line contain string.
* `"!string"` - inverse of above, checks that no line contains string
* `"\\!string"` - escape sequence, check if any line contains `"!string"`
* `"/regex/"` - verifies that line contains regex
* `"!/regex/"` - inverse of above, checks that no line contains regex

**NOTE:** Pattern attributes do not support [Advanced Matchers](#advanced-matchers)

**NOTE:** Regex support is based on golang's regex engine documented [here](https://golang.org/pkg/regexp/syntax/)

**NOTE:** You will **need** the double backslash (`\\`) escape for Regex special entities, for example `\\s` for blank spaces.

### Example
```bash
$ cat /tmp/test.txt
found
!alsofound


$ cat goss.yaml
file:
  /tmp/test.txt:
    exists: true
    contains:
    - found
    - /fou.d/
    - "\\!alsofound"
    - "!missing"
    - "!/mis.ing/"

$ goss validate
..

Total Duration: 0.001s
Count: 2, Failed: 0
```

## Advanced Matchers
Goss supports advanced matchers by converting json input to [gomega](https://onsi.github.io/gomega/) matchers.

### Examples

Validate that user `nobody` has a `uid` that is less than `500` and that they are **only** a member of the `nobody` group.

```yaml
user:
  nobody:
    exists: true
    uid:
      lt: 500
    groups:
      consist-of: [nobody]
```

Matchers can be nested for more complex logic, for example you can ensure that you have 3 kernel versions installed and none of them are `4.1.0`:

```yaml
package:
  kernel:
    installed: true
    versions:
      and:
        - have-len: 3
        - not:
            contain-element: "4.1.0"
```

For more information see:
* [gomega_test.go](https://github.com/aelsabbahy/goss/blob/master/resource/gomega_test.go) - For a complete set of supported json -> Gomega mapping
* [gomega](https://onsi.github.io/gomega/) - Gomega matchers reference

## Templates

Goss test files can leverage golang's [text/template](https://golang.org/pkg/text/template/) to allow for dynamic or conditional tests.

Available variables:
* `{{.Env}}`  - Containing environment variables
* `{{.Vars}}` - Containing the values defined in [--vars](#global-options) file

Available functions beyond text/template [built-in functions](https://golang.org/pkg/text/template/#hdr-Functions):
* `mkSlice` - Retuns a slice of all the arguments. See examples below for usage.

**NOTE:** gossfiles containing text/template `{{}}` controls will no longer work with `goss add/autoadd`. One way to get around this is to split your template and static goss files and use [gossfile](#gossfile) to import.

### Examples

Using [puppetlabs/facter](https://github.com/puppetlabs/facter) or [chef/ohai](https://github.com/chef/ohai) as external tools to provide vars.
```bash
$ goss --vars <(ohai) validate
$ goss --vars <(facter -j) validate
```

Using `mkSlice` to define a loop locally.
```yaml
file:
{{- range mkSlice "/etc/passwd" "/etc/group"}}
  {{.}}:
    exists: true
    mode: "0644"
    owner: root
    group: root
    filetype: file
{{end}}
```

Using Env variables and a vars file:

**vars.yaml:**
```yaml
centos:
  packages:
    kernel:
      - "4.9.11-centos"
      - "4.9.11-centos2"
debian:
  packages:
    kernel:
      - "4.9.11-debian"
      - "4.9.11-debian2"
users:
  - user1
  - user2
```

**goss.yaml:**
```yaml
package:
# Looping over a variables defined in a vars.yaml using $OS environment variable as a lookup key
{{range $name, $vers := index .Vars .Env.OS "packages"}}
  {{$name}}:
    installed: true
    versions:
    {{range $vers}}
      - {{.}}
    {{end}}
{{end}}

# This test is only when OS=centos variable is defined
{{if eq .Env.OS "centos"}}
  libselinux:
    installed: true
{{end}}

# Loop over users
user:
{{range .Vars.users}}
  {{.}}:
    exists: true
    groups:
    - {{.}}
    home: /home/{{.}}
    shell: /bin/bash
{{end}}
```

Rendered results:
```bash
# To validate:
$ OS=centos goss --vars vars.yaml validate
# To render:
$ OS=centos goss --vars vars.yaml render
# To render with debugging enabled:
$ OS=centos goss --vars vars.yaml render --debug
```
