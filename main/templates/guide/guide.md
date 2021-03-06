So after wetting your appetite in the [Tutorial]({{url_for('tutorial')}}) we
will now give you the big picture overview of the gae-init environment. After
reading this guide you should understand how the individual blocks of gae-init
play together, how you can benefit most of them and where to start in case you
want to customise things.

{% raw %}

Overview {#overview}
--------

- [Interplay of components](#interplay)
    - [run.py](#run_py)
    - [config.py](#config_py)
    - [appcfg & app.yaml](#appengine)
    - [server side tasks](#server_side_tasks) (setup, compiling, minification, bundling)
        - [Node.js](#node)
        - [Grunt](#grunt)
        - [Bower](#bower)
        - [livereload](#livereload)
        - [CoffeeScript](#coffeescript)
        - [Less](#less)
    - [server side libs](#server_side_libs) (Python)
    - [client side libs](#client_side_libs) (JS)
- [main.py](#main_py)
- [templates/base.html](#templates_base)
    - [available blocks](#templates_blocks)
    - [bits](#templates_bits) (especially style + js)
- [Frequent tasks / dev workflows](#workflows)
    - [running the dev-server](#run_dev_server)
    - [customize styles](#custom_styles)
    - [custom JavaScript / CoffeeScript](#custom_js)
    - [add a new Python lib](#new_python_lib)
    - [add a new / update/reinstall JS lib(s)](#new_js_lib) (Bower)
    - [add a new page](#new_page)


Interplay of components {#interplay}
--------------------------------------
gae-init is built on top of several tools that try to ease your job as developer
by supporting common tasks. In order to be awesome you have to know these little
helpers and how they work together, so let's get you on the line.

### run.py {#run_py}
It all starts with `./run.py` which you might still know from the
[Tutorial]({% endraw %}{{url_for('tutorial')}}{% raw %}). As its name implies it
runs your dev-server on [http://localhost:8080](http://localhost:8080) with
`./run.py -s`.

What we didn't mention in the Tutorial is that you typically want start a second
`./run.py -w` in a different terminal window to watch for changes in `*.coffee`
and `*.less` files and automagically trigger a recompilation (also see our
Frequent Tasks section [running the dev-server](#run_dev_server) for some more
included batteries).

`./run.py` can also flush your dev-server's datastore `-f`, so you can start
from scratch with a fresh store without worrying about interference with already
existing objects in it from previous runs.

`./run.py -m` will minify and bundle together your `*.js` and `*.css` files
(manually generated ones, the ones from installed libs as well as the ones
generated from CoffeeScript and Less). On your dev-server you will normally not
see the minified version of your page as it is cumbersome for debugging, but in
production the minification can drastically reduce the size and amounts of
requests to your page (in case of this docs page from 49 to 28) and thereby
increase your page load speed. Therefore, you need to run `./run.py -m` before
deploying your app. If you want to see the effects without deploying feel free
to temporarily set `PRODUCTION=True` in your `config.py`. Be aware of the fact
that the minified files don't auto-update though, so you need to re-run the
minification if you change something in your scripts or styles.

Last but not least `./run.py -C` will clean up and reinstall all the local
tools and libraries that were downloaded via Node and Bower to their latest
versions. This should be used whenever you want to update your 3rd party
libraries and tools. See also the [Node.js](#node), [Bower](#bower) and
[add a new / update/reinstall js lib(s)](#new_js_lib) sections below.

For completeness also have a look at our
[Run Script]({% endraw %}{{url_for('run')}}{% raw %}) page.


### config.py {#config_py}
`config.py` is a central place to orchestrate the gae-init toolbox.

`run.py` for example needs to know a couple of things for its various tasks
like minification and compiling: which are the relevant Less, JavaScript and
CoffeeScript files. `run.py` takes these from `config.py` which defines
`SCRIPTS` and `STYLES`. The same `SCRIPTS` are used by some magic in
[/main/templates/bit/script.html](https://github.com/gae-init/gae-init-docs/blob/master/main/templates/bit/script.html)
to load the minified and bundled JavaScript in production as well as the actual
source files on the dev-server.
This means that if you want to make use of the minification feature, be sure
to add your javascript files to the `SCRIPTS` section in `config.py`. This has
the added benefit that you don't need to manually link them in your HTML, but
that the script bit can do its magic. (See the
[API reference on config.py]({% endraw %}{{url_for('reference')}}{% raw %}#config.STYLES)
as well as [add a new js lib](#new_js_lib) and [template bits](#templates_bits)
below for further information.)

Apart from that `config.py` also sets some other important global constants
like `PRODUCTION` which is `true` if deployed and run on the actual Google App
Engine infrastructure and `DEVELOPMENT` and `DEBUG` which are `true` when run
on the dev-server. `config.py` also extracts version information and the
application id into `CURRENT_VERSION_ID`, `CURRENT_VERSION_NAME`,
`CURRENT_VERSION_TIMESTAMP`, `CURRENT_VERSION_DATE` and `APPLICATION_ID` (see
[API reference on config.py]({% endraw %}{{url_for('reference')}}{% raw %}#config)
for more).

`CONFIG_DB` and `SECRET_KEY` encapsulate the datastore's singleton config
object as well as the secret key which is randomly autogenerated on first run
of your app and used in Flask for cookie protection (session cookies and auth).

`DEFAULT_DB_LIMIT` is used for the default paging size in lists (see
[utils.retrieve_dbs](https://github.com/gae-init/gae-init-docs/blob/master/main/util.py#L52)
for more on this).


### appcfg & app.yaml {#appengine}
As every other App Engine app you will find the main configurations (e.g., the
app's name, its programming language, the main handler definitions, ...) in
[main/app.yaml](https://github.com/gae-init/gae-init-docs/blob/master/main/app.yaml).

The main handler definitions include serving the `main/static` dir under `/p/`
and invoke `main/main.app` (so the app object defined in
[main/main.py](https://github.com/gae-init/gae-init-docs/blob/master/main/main.py)
for nearly everything else.

`app.yaml` also defines `skip_files`, so what not to upload. Notice how
`lib/.*`, `static/dst/.*`, some folders from `static/ext/` and `static/src/.*`
are excluded. The reason for these to be excluded is the minification process
as described for `./run.py -m` above and configured in `config.py`. This means
that before deployment you need ot run `./run.py -m` as your production server
will otherwise lack critical libraries at runtime. This also means that you
should not place own libraries or files in these folders manually without
adding them to `config.py`.

After running `./run.py -m` you can deploy the app with Google App Engine's
tools as usual, either from command line with `appcfg.py` or via the
GoogleAppEngineLauncher app.



### server side tasks (setup, compiling, minification, bundling) {#server_side_tasks}

#### Node.js {#node}

#### Grunt {#grunt}

#### Bower {#bower}

#### livereload {#livereload}

#### CoffeeScript {#coffeescript}

#### Less {#less}

### server side libs (Python) {#server_side_libs}
Python libraries are in general installed with pip via the
[requirements.txt](https://github.com/gae-init/gae-init-docs/blob/master/requirements.txt)
file in the basedir. You can specify necessary libraries in it. The next run of
`./run.py -s` will install new libaries into a virtual environment in the
`temp/venv` folder and then copy them into `main/lib`.

[appengine_config.py](https://github.com/gae-init/gae-init-docs/blob/master/main/appengine_config.py)
will take care that the `sys.path` is modified at runtime to include these libs
so they can simply be imported. On the dev-server the lib folder is added to
the path as is (`sys.path.insert(0, 'lib')`), but in production the lib folder
is zipped by `run.py -m` ([see above](#run_py)) and so all libraries are
imported from one single file called `lib.zip`.

In case you need to include libraries which do not work when zipped you can
create a `main/libx` folder and manually copy those libraries to it. The folder
is included in the `sys.path` on both, the development server and in
production. Don't manually copy libraries into the `main/lib` folder as it is
excluded in [app.yaml](#appengine) and won't be uploaded to App Engine (as its
contents are uploaded in a zipped form as described above).



### client side libs (js) {#client_side_libs}



main.py {#main_py}
--------------------


templates/base.html {#templates_base}
---------------------------------------

### available blocks {#templates_blocks}

### bits (especially style + js) {#templates_bits}



Frequent tasks / dev workflows {#workflows}
---------------------------------------------

### running the dev-server {#run_dev_server}
You can simply run the server from the project directory (one above main) by
executing `run.py` in a terminal with the `-s` flag.
```bash
$ cd /path/to/project-name
$ ./run.py -s
```
Not only will this start the server
[http://localhost:8080/](http://localhost:8080/), but it will also
make sure that all dependencies are checked and install them if necessary.
Furthermore, it will trigger an initial compilation of the CoffeeScript and Less
files to JavaScript and CSS files so your browser has some JavaScript and
stylesheets when visiting the dev webserver.

While this should get your basic server up and running, in order not to have to
restart it manually whenever you for example change some CoffeeScript
(`*.coffee`) or Less (`*.less`) files, we suggest that run two more commands at
the same time in separate terminals:
```bash
$ ./run.py -w
```
This will watch CoffeeScript and Less files and trigger a recompilation whenever
they are changed.

And:
```bash
$ ./node_modules/.bin/grunt
```
This will provide a watch-daemon which thanks to a the `livereload.js` which is
embedded in the development mode will automatically reload the pages in your
browser window when any of the relevant files change. Cool, eh? ;)

If the path to grunt seems a bit weird: you can also safely install it globally:
```bash
$ npm install -g grunt-cli
```
After this you can just run grunt from the project root:
```bash
$ grunt
```


### customize styles {#custom_styles}
In order to customize the style of your page you can edit or in most cases
simply add further files in
[main/static/src/style](https://github.com/gae-init/gae-init-docs/blob/master/main/static/src/style),
the main file being
[style.less](https://github.com/gae-init/gae-init-docs/blob/master/main/static/src/style/style.less).
If you add further files make sure to include them in `style.less`.

[Less](http://lesscss.org) is a powerful CSS generation language that will help
you to not repeat yourself while writing your stylesheets. In the case of
`style.less` you will mainly find it including other files. Have a look at them
and at our section about [templates/base.html](#templates_base) in order to
understand what's going on.



### custom JavaScript / CoffeeScript {#custom_js}

### add a new Python lib {#new_python_lib}

### add a new / update/reinstall js lib(s) (bower) {#new_js_lib}

### add a new page {#new_page}

{% endraw %}
