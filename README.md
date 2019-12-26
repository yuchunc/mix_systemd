# mix_systemd

This library generates a
[systemd](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
unit file to manage an Elixir release. It supports releases generated by Elixir 1.9+
[Mix](https://hexdocs.pm/mix/Mix.Tasks.Release.html) and
[Distillery](https://hexdocs.pm/distillery/home.html).

It uses standard systemd functions and conventions to make your app a more
"native" OS citizen, and takes advantage of systemd features to improve
security and reliability.

At its heart, it's a mix task which reads information about the project from
`mix.exs` and `config/config.exs` then generates systemd unit files using Eex
templates.

The goal is that the project defaults will generate a good systemd unit file,
and standard options support more specialized use cases.

While it can be used standalone, more advanced use cases use scripts
from e.g. [mix_deploy](https://github.com/cogini/mix_deploy).

## Installation

Add `mix_systemd` to `deps` in `mix.exs`:

```elixir
{:mix_systemd, "~> 0.7.0"},
```

## Configuration

The library tries to choose smart defaults, so you may not need to configure
anything.

The library reads the app name from `mix.exs` and calculates default values
for its configuration parameters. For example, if your app is nomed `foo_bar`,
it will create a service named `foo-bar`, deployed to `/srv/foo-bar`, running
under the user `foo-bar`.

You can override these parameters using settings in `config/config.exs`, e.g.:

```elixir
config :mix_systemd,
    app_user: "app",
    app_group: "app",
    base_dir: "/opt",
    env_vars: [
        "PORT=8080",
    ]
```

Following is a more secure config which deploys the app using a
different user account from what the app runs under, so the
release source dirs are readonly to the app. It runs a script on startup which
pulls app config from S3 in TOML format, putting it in the `/etc/app` dir. It
uses a
[config provider](https://hexdocs.pm/elixir/Config.Provider.html) to load files
in [TOML](https://hexdocs.pm/toml_config/readme.html), which needs to write
a temp file to `/run/app`. It sets `runtime_directory_preserve` to `yes`
to help in debugging startup issues.

```elixir
config :mix_systemd,
  app_user: "app",
  app_group: "app",
  exec_start_pre: [
    "!/srv/app/bin/deploy-sync-config-s3"
  ],
  dirs: [
    :runtime,       # App runtime files which may be deleted between runs, /run/#{ext_name}
    :configuration, # App configuration, e.g. db passwords, /etc/#{ext_name}
    # :state,         # App data or state persisted between runs, /var/lib/#{ext_name}
    # :cache,         # App cache files which can be deleted, /var/cache/#{ext_name}
    # :logs,          # App external log files, not via journald, /var/log/#{ext_name}
    # :tmp,           # App temp files, /var/tmp/#{ext_name}
  ],
  runtime_directory_preserve: "yes",
  env_vars: [
    {"RELEASE_TMP", :runtime_dir},
  ]
```

Here is [a complete example app which uses mix_deploy](https://github.com/cogini/mix-deploy-example).

## Usage

First, use the `systemd.init` task to template files from the library to the
`rel/templates/systemd` directory in your project.

```shell
mix systemd.init
```

Next, generate output files in the build directory under
`_build/#{mix_env}/systemd/lib/systemd/system`.

```shell
MIX_ENV=prod mix systemd.generate
```

## Configuration options

The following sections describe configuration options. See
`lib/mix/tasks/systemd.ex` for all the details.

If you need something special, then you can modify the templates in
`rel/templates/systemd` and check them into source control. Patches are
welcome!

### Basics

`app_name`: Elixir application name, an atom, from the `app` field in the `mix.exs` project.

`module_name`: Elixir camel case module name version of `app_name`.

`release_name`: Name of release, default `app_name`.

`ext_name`: External name, used for files and directories.
Default is `app_name` with underscores converted to "-".

`service_name`: Name of the systemd service, default `ext_name`.

`app_user`: OS user account that the app runs under, default `ext_name`.

`app_group`: OS group account, default `ext_name`.

### Directories

`base_dir`: Base directory for app files on the target, default `/srv`.

`deploy_dir`: Directory for app files on the target, default `#{base_dir}/#{ext_name}`

We use the [standard app directories](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RuntimeDirectory=),
for modern Linux systems. App files under `/srv`, configuration under
`/etc`, transient files under `/run`, data under `/var/lib`.

Directories are named based on the app name, e.g. `/etc/#{ext_name}`.
The `dirs` variable specifies which directories the app uses.
By default, it doesn't set up anything. To enable them, configure the `dirs`
param, e.g.:

```elixir
dirs: [
  :runtime,       # App runtime files which may be deleted between runs, /run/#{ext_name}
  :configuration, # App configuration, e.g. db passwords, /etc/#{ext_name}
  # :state,         # App data or state persisted between runs, /var/lib/#{ext_name}
  # :cache,         # App cache files which can be deleted, /var/cache/#{ext_name}
  # :logs,          # App external log files, not via journald, /var/log/#{ext_name}
  # :tmp,           # App temp files, /var/tmp/#{ext_name}
],
```

Recent versions of systemd (since 235) will create these directories at
start time based on the settings in the unit file. With earlier systemd
versions, create them beforehand using installation scripts, e.g.
[mix_deploy](https://github.com/cogini/mix_deploy).

For security, we set permissions more restrictively than the systemd defaults.
You can configure them by setting variables such as `configuration_directory_mode`.
See the defaults in `lib/mix/tasks/systemd.ex`.

`systemd_version`: Sets the systemd version on the target system, default 235.
This determines which systemd features the library will enable. If you are
targeting an older OS release, you may need to change it. Here are the systemd
versions in common OS releases:

* CentOS 7: 219
* Ubuntu 16.04: 229
* Ubuntu 18.04: 237

`release_system`: `:mix | :distillery`, default `:mix`

Identifies the system which was used to generate the release,
[Mix](https://hexdocs.pm/mix/Mix.Tasks.Release.html) or
[Distillery](https://hexdocs.pm/distillery/home.html).
This configures the command which starts the system and some environment vars.

### Directory structure

The library uses a directory structure under `deploy_dir` which supports
multiple releases, similar to [Capistrano](https://capistranorb.com/documentation/getting-started/structure/).

* `scripts_dir`: deployment scripts which e.g. start and stop the unit, default `bin`.
* `current_dir`: where the current Erlang release is unpacked or referenced by symlink, default `current`.
* `releases_dir`: where versioned releases are unpacked, default `releases`.
* `flags_dir`: dir for flag files to trigger restart, e.g. when `restart_method` is `:systemd_flag`, default `flags`.

When using multiple releases and symlinks, the deployment process works as follows:

1. Create a new directory for the release with a timestamp like
   `/srv/foo/releases/20181114T072116`.

2. Upload the new release tarball to the server and unpack it to the releases dir

3. Make a symlink from `/srv/#{ext_name}/current` to the new release dir.

4. Restart the app.

If you are only keeping a single version, then deploy it to
the directory `/srv/#{ext_name}/current`.

### Environment vars

The library sets env vars in the unit file:

* `MIX_ENV`: `mix_env`, default `Mix.env()`
* `LANG`: `C.UTF-8`

* `RUNTIME_DIR`: `runtime_dir`, if `:runtime` in `dirs`
* `CONFIGURATION_DIR`: `configuration_dir`, if `:configuration` in `dirs`
* `LOGS_DIR`: `logs_dir`, if `:logs` in `dirs`
* `CACHE_DIR`: `cache_dir`, if `:cache` in `dirs`
* `STATE_DIR`: `state_dir`, if `:state` in `dirs`
* `TMP_DIR`: `tmp_dir`, if `:tmp` in `dirs`

You can set additional vars using `env_vars`, e.g.:

```elixir
env_vars: [
    "PORT=8080",
]
```
You can also reference the value of other parameters by name, e.g.:

```elixir
env_vars: [
    {"RELEASE_TMP", :runtime_dir},
]
```

The unit file reads environment vars from a series of files:

* `etc/environment` within the release, e.g. `/srv/app/current/etc/environment`
* `#{deploy_dir}/etc/environment`, e.g. `/srv/app/etc/environment`
* `#{configuration_dir}/environment`, e.g. `/etc/app/environment`
* `#{runtime_dir}/environment`, e.g. `/run/app/environment`

These files are optional, the system will start without them. Later values
override earlier values, so you can set defaults in the release which get
overridden in the deployment or runtime environment.

`etc/environment` in the release is only enabled when using Distillery. You can
set the files with an overlay in `rel/config.exs`, e.g.:

```elixir
environment :prod do
  set overlays: [
    {:mkdir, "etc"},
    {:copy, "rel/etc/environment", "etc/environment"},
    # {:template, "rel/etc/environment", "etc/environment"}
  ]
end
```

## Runtime dirs

At a minimum, the release scripts need to be able to write startup logs and
generate the application config.

By default, they write logs to `logs` directory under the release, e.g.
`/srv/foo/current/logs` and config temp files to `/srv/foo/current/tmp` dir.

For added security, you can deploy the app using a different user account from
the one that the app runs under. In that case, you need to either make these
directories writable by the app user or set an environment variable
to tell the release where it can write its files. For Mix releases, that is
`RELEASE_TMP` and for Distillery it is `RELEASE_MUTABLE_DIR`, e.g.:

```elixir
env_vars: [
    {"RELEASE_TMP", :runtime_dir},
]
```

By default systemd will delete the runtime directory when restarting the app,
which can be annoying when debugging startup issues. You can set
`runtime_directory_preserve` to `restart` or `yes` (see
[RuntimeDirectoryPreserve](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RuntimeDirectoryPreserve=)).
You can also set `RELEASE_TMP/RELEASE_MUTABLE_DIR` to `/tmp` or the application
`:tmp_dir` so that you can see what is happening.

### Systemd and OS

The following variables set systemd variables:

`service_type`: `:simple | :exec | :notify | :forking`. systemd
[Type](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=), default `:simple`.

Modern applications don't fork, they run in the foreground and
rely on the supervisor to manage them as a daemon. This is done by setting
`service_type` to `:simple` or `:exec`. Note that in `simple` mode, systemd
doesn't actually check if the app started successfully, it just continues
starting other units. If something depends on your app being up, `:exec` may be
better.

Set `service_type` to `:forking`, and the library sets `pid_file` to
`#{runtime_directory}/#{app_name}.pid` and sets the `PIDFILE` env var to tell
the boot scripts where it is.

The Erlang VM runs pretty well in foreground mode, but traditionally runs as
as a standard Unix-style daemon, so forking might be better. Systemd
expects foregrounded apps to die when their pipe closes. See
https://elixirforum.com/t/systemd-cant-shutdown-my-foreground-app-cleanly/14581/2

`restart_method`: `:systemctl | :systemd_flag`. Default `:systemctl`

Set this to `:systemd_flag`, and the library will generate an additional
unit file which watches for changes to a flag file and restarts the
main unit. This allows updates to be pushed to the target machine by an
unprivileged user account which does not have permissions to restart
processes. Touch the file `#{flags_dir}/restart.flag` and systemd will
restart the unit.

`working_dir`: Current working dir for app. systemd
[WorkingDirectory](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#WorkingDirectory=),
default `current_dir`.

`limit_nofile`: Limit on open files, systemd
[LimitNOFILE](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#LimitCPU=),
default 65535.

`umask`: Process umask, systemd
[UMask](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#UMask=),
default "0027". Note that this is octal, so it needs to be a string.

`restart_sec`: Time in seconds to wait between restarts, systemd
[RestartSec](https://www.freedesktop.org/software/systemd/man/systemd.service.html#RestartSec=),
default 100ms.

`syslog_identifier`: Logging name, systemd
[SyslogIdentifier](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SyslogIdentifier=),
default `service_name`

### Runtime configuration

For configuration, we use a combination of build time settings, deploy
time settings, and runtime settings.

The settings in `config/config.exs` and `config/prod.exs` are baked into the release.

We can extend them with additional configuration such as DB logins which are stored
outside the release in the configuration dir `/etc/#{ext_name}`.
The app reads these config files on startup and merges them into the app config.

In on-premises deployments, we might generate the machine-specific
configuration once when setting up the app. In cloud and other dynamic
environments, we may run from a read-only image, e.g. an Amazon AMI, which gets
configured at start up based on the environment by copying the config from an
S3 bucket or a configuration store like [AWS Systems Manager Parameter
Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html)
or etcd.

Some things change dynamically each time the app starts, e.g. the IP address of
the machine, or periodically, such as AWS access keys in an IAM instance role.

#### Coniguration providers

With [Mix releases](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-runtime-configuration),
you can create a `config/releases.exs` file in Elixir format which has e.g. db
secrets. Or you can add a [TOML config file](https://hexdocs.pm/toml_config/readme.html)
which is loaded by a config provider.

With Distillery, configure providers in `rel/config.exs`:

```elixir
environment :prod do
  set config_providers: [
    {Mix.Releases.Config.Providers.Elixir, ["${CONFIGURATION_DIR}/config.exs"]}
  ]
end
```
or

```elixir
environment :prod do
  set config_providers: [
    {Toml.Provider, [path: "${CONFIGURATION_DIR}/config.toml"]},
  ]
end
```

Add the TOML config provier to `mix.exs`:

```elixir
{:toml_config_provider, "~> 0.2.0"}
```

Configure `mix_systemd` to set the environment var `REPLACE_OS_VARS=true`, and
the startup scripts will expand the `CONFIGURATION_DIR` env var at runtime and load the file.


This library supports three ways to get runtime config:

#### `ExecStartPre` scripts

Scripts specified in `exec_start_pre` (systemd
[ExecStartPre](https://www.freedesktop.org/software/systemd/man/systemd.service.html#ExecStartPre=)])
run before the main `ExecStart` script runs, e.g.:

```elixir
exec_start_pre: [
"!/srv/foo/bin/deploy-sync-config-s3"
]
```

This runs the `deploy-sync-config-s3` script from `mix_deploy`, which
copies config files from an S3 bucket into `/etc/foo`. By default,
scripts run as the same user and group as the main script. Putting
`!` in front makes the script run with
[elevated privileges](https://www.freedesktop.org/software/systemd/man/systemd.service.html#ExecStart=),
allowing it to write config to `/etc/foo` even if the main user account cannot for security reasons.

In Elixir 1.9+ releases you can use `rel/env.sh.eex`, but this runs earlier
with elevated permissions, so it is useful as well.

#### ExecStart wrapper script

Instead of running the main `ExecStart` script directly, you can run a shell script
which sets up the environment, then runs the main script with `exec`.
Set `exec_start_wrap` to the name of the script, e.g.
`deploy-runtime-environment-wrap` from `mix_deploy`.

In Elixir 1.9+ releases you can use `rel/env.sh.eex`, but this runs earlier
with elevated permissions, so it may be useful as well.

#### Runtime environment service

You can run your own separate service to configure the runtime environment
before the app runs.  Set `runtime_environment_service_script` to a script such
as `deploy-runtime-environment-file` from `mix_deploy`. This library will
create a `#{service_name}-runtime-environment.service` unit and make it a
systemd runtime dependency of the app.

### Runtime dependencies

Systemd starts units in parallel when possible. To enforce ordering, set
`unit_after_targets` to the names of systemd units that this unit depends on.
For example, if this unit should run after cloud-init to get [runtime network
information](https://cloudinit.readthedocs.io/en/latest/topics/network-config.html#network-configuration-outputs),
set:

```elixir
unit_after_targets: [
    "cloud-init.target"
]
```

## Security

`paranoia`: Enable systemd security options, default `false`.

    NoNewPrivileges=yes
    PrivateDevices=yes
    PrivateTmp=yes
    ProtectSystem=full
    ProtectHome=yes
    PrivateUsers=yes
    ProtectKernelModules=yes
    ProtectKernelTunables=yes
    ProtectControlGroups=yes
    MountAPIVFS=yes
                                                                                                    │
`chroot`: Enable systemd [chroot](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RootDirectory=), default `false`.
Sets systemd `RootDirectory` is set to `current_dir`. You can also set systemd [ReadWritePaths=, ReadOnlyPaths=,
InaccessiblePaths=](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ReadWritePaths=)
with the `read_write_paths`, `read_only_paths` and `inaccessible_paths` vars, respectively.
