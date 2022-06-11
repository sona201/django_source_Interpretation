启动命令
```
python3 manage.py runserver 0.0.0.0:8000
```

平时用的命令就是这个, 入口是 `manage.py` 文件

#### manage.py 管理入口文件
```python
def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'firstDjango.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)
```

manage.py 先配置环境变量, 执行命令后会把参数 `runserver 0.0.0.0:8000` 作为系统参数传给函数 `execute_from_command_line`

#### site-packages/django/core/management/__init__.py

```python
def execute_from_command_line(argv=None):
    """Run a ManagementUtility."""
    utility = ManagementUtility(argv)
    utility.execute()
```

该函数仅仅是处理命令的方法, 实际做的就是实例化 `ManagementUtility()` , 调用实例的 `execute()` 方法
`ManagementUtility` 和 `execute_from_command_line` 都在 management 的 __init__ 下

```python
    def execute(self):
        """
        Given the command-line arguments, figure out which subcommand is being
        run, create a parser appropriate to that command, and run it.
        """
        try:
            subcommand = self.argv[1]
        except IndexError:
            subcommand = "help"  # Display help if no arguments were given.

        # Preprocess options to extract --settings and --pythonpath.
        # These options could affect the commands that are available, so they
        # must be processed early.
        parser = CommandParser(
            prog=self.prog_name,
            usage="%(prog)s subcommand [options] [args]",
            add_help=False,
            allow_abbrev=False,
        )
        parser.add_argument("--settings")
        parser.add_argument("--pythonpath")
        parser.add_argument("args", nargs="*")  # catch-all
        try:
            options, args = parser.parse_known_args(self.argv[2:])
            handle_default_options(options)
        except CommandError:
            pass  # Ignore any option errors at this point.

        try:
            settings.INSTALLED_APPS
        except ImproperlyConfigured as exc:
            self.settings_exception = exc
        except ImportError as exc:
            self.settings_exception = exc

        if settings.configured:
            # Start the auto-reloading dev server even if the code is broken.
            # The hardcoded condition is a code smell but we can't rely on a
            # flag on the command class because we haven't located it yet.
            if subcommand == "runserver" and "--noreload" not in self.argv:
                try:
                    autoreload.check_errors(django.setup)()
                except Exception:
                    # The exception will be raised later in the child process
                    # started by the autoreloader. Pretend it didn't happen by
                    # loading an empty list of applications.
                    apps.all_models = defaultdict(dict)
                    apps.app_configs = {}
                    apps.apps_ready = apps.models_ready = apps.ready = True

                    # Remove options not compatible with the built-in runserver
                    # (e.g. options for the contrib.staticfiles' runserver).
                    # Changes here require manually testing as described in
                    # #27522.
                    _parser = self.fetch_command("runserver").create_parser(
                        "django", "runserver"
                    )
                    _options, _args = _parser.parse_known_args(self.argv[2:])
                    for _arg in _args:
                        self.argv.remove(_arg)

            # In all other cases, django.setup() is required to succeed.
            else:
                django.setup()

        self.autocomplete()

        if subcommand == "help":
            if "--commands" in args:
                sys.stdout.write(self.main_help_text(commands_only=True) + "\n")
            elif not options.args:
                sys.stdout.write(self.main_help_text() + "\n")
            else:
                self.fetch_command(options.args[0]).print_help(
                    self.prog_name, options.args[0]
                )
        # Special-cases: We want 'django-admin --version' and
        # 'django-admin --help' to work, for backwards compatibility.
        elif subcommand == "version" or self.argv[1:] == ["--version"]:
            sys.stdout.write(django.get_version() + "\n")
        elif self.argv[1:] in (["--help"], ["-h"]):
            sys.stdout.write(self.main_help_text() + "\n")
        else:
            self.fetch_command(subcommand).run_from_argv(self.argv)
```
截取了 ManagementUtility 的 execute 方法代码
这个就是 `CommandParser` 对命令的解析, 优先加载django的配置
最重要的是最后一句 `self.fetch_command(subcommand).Command(self.argv)` 
执行 `fetch_command(subcommand)` 得到了 'django.contrib.staticfiles.management.commands.runserver' 结果
对应的文件路径: `site-packages/django/contrib/staticfiles/management/commands/runserver.py`

#### site-packages/django/core/management/commands/runserver.py

执行函数导入
```python
def load_command_class(app_name, name):
    """
    Given a command name and an application name, return the Command
    class instance. Allow all errors raised by the import process
    (ImportError, AttributeError) to propagate.
    """
    module = import_module("%s.management.commands.%s" % (app_name, name))
    return module.Command()
```
即 将 `c` 模块导入
然后实例化 django.contrib.staticfiles.management.commands.runserver.Command()
所有的命令方法都是有 command 类

这个函数就是获取对应文件名, 然后导入Command类, 执行 `run_from_argv` 方法

django.contrib.staticfiles.management.commands.runserver 中没有 run_from_argv 方法
django.core.management.commands.runserver 中也没有 run_from_argv 方法
django.core.management.base.BaseCommand 中有 run_from_argv 方法 主要是执行 self.execute 方法

django.core.management.commands.runserver 的execute方法

```python
    def execute(self, *args, **options):
        if options["no_color"]:
            # We rely on the environment because it's currently the only
            # way to reach WSGIRequestHandler. This seems an acceptable
            # compromise considering `runserver` runs indefinitely.
            os.environ["DJANGO_COLORS"] = "nocolor"
        super().execute(*args, **options)
```

django.core.management.base.BaseCommand  的execute 方法
```python
    def execute(self, *args, **options):
        """
        Try to execute this command, performing system checks if needed (as
        controlled by the ``requires_system_checks`` attribute, except if
        force-skipped).
        """
        if options["force_color"] and options["no_color"]:
            raise CommandError(
                "The --no-color and --force-color options can't be used together."
            )
        if options["force_color"]:
            self.style = color_style(force_color=True)
        elif options["no_color"]:
            self.style = no_style()
            self.stderr.style_func = None
        if options.get("stdout"):
            self.stdout = OutputWrapper(options["stdout"])
        if options.get("stderr"):
            self.stderr = OutputWrapper(options["stderr"])

        if self.requires_system_checks and not options["skip_checks"]:
            if self.requires_system_checks == ALL_CHECKS:
                self.check()
            else:
                self.check(tags=self.requires_system_checks)
        if self.requires_migrations_checks:
            self.check_migrations()
        output = self.handle(*args, **options)
        if output:
            if self.output_transaction:
                connection = connections[options.get("database", DEFAULT_DB_ALIAS)]
                output = "%s\n%s\n%s" % (
                    self.style.SQL_KEYWORD(connection.ops.start_transaction_sql()),
                    output,
                    self.style.SQL_KEYWORD(connection.ops.end_transaction_sql()),
                )
            self.stdout.write(output)
        return output
```

执行 self.handle 方法
django.core.management.commands.runserver 的 handle 方法

django.core.management.commands.runserver 的 run 方法
django.core.management.commands.runserver 的 inner_run 方法

```python
            handler = self.get_handler(*args, **options)
            run(
                self.addr,
                int(self.port),
                handler,
                ipv6=self.use_ipv6,
                threading=threading,
                server_cls=self.server_cls,
            )
```