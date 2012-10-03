Project Eideticker
==================

Project Eideticker is an automated test harness that captures and analyzes
browser output via HDMI. It currently only supports various browsers (Fennec,
Stock) on Android, but support for other platforms is planned. There is
experimental support for B2G on the pandaboard.

### Requirements

* Linux system with zip, ffmpeg, g++, python 2.7 development and virtualenv
  installed. On Ubuntu, you can get this setup by running:

    `sudo apt-get install -y zip ffmpeg g++ python2.7-dev python-virtualenv`

* Blackmagic Design DeckLink HD Extreme 3D card and the appropriate drivers if
  you want to actually capture video (it is possible to use the Eideticker
  harness without the video capture step, which can be useful for certain
  cases: see below).

* Installed Android SDK with the tools in your path. Please follow the
  instructions at https://wiki.mozilla.org/Mobile/Fennec/Android#Setup_a_Build_Environment

* A supported mobile phone running Android with Orangutan installed in
  `/system/ Eideticker's tests are specifically tuned for specific screen
  dimensions. Currently only the LG G2X (running in portrait mode) and the
  Galaxy Nexus (running in landscape) are supported. Your device must also be
  rooted and, if using SUTAgent, you must configure the su binary to
  automatically allow SUTAgent to run commands as root in silent mode (so that
  notifications don't pop up while running tests). For information on installing
  Orangutan, see that project's README at http://github.com/wlach/orangutan.

### Installation

Run `bootstrap.sh` in the root directory to set everything up.

### Usage

Eideticker is meant to be run in a virtualenv, so the first step is to set
that up in your shell by running this command within the root directory:

    source ./bin/activate

The next step depends on whether you're using ADB or SUTAgent to interface
with the device. If the former, just connect your Android device to your
computer's USB port. That's it, you should now be set to run tests!

If you're using SUTAgent, you'll want to start the SUTAgent app, note the
ip address of your phone, then set the following environment variables:

    export DM_TRANS=sut
    export TEST_DEVICE=<device ip of phone>

If you are using a B2G pandaboard with adb, then you'll need sudo privileges for adb:

    adb kill-server
    sudo adb start-server

#### Console Mode

Console mode is meant for internal profiling. You run a program
called get-metric-for-build.py with a specific test against an Android apk,
and out will pop several results. This is not yet supported for B2G.

    ./bin/get-metric-for-build.py --use-apks <test> <apk of build 1> [apk of build 2] ...

For example, to run the canvas clock example against Fennec nightly, try
this:

    ./bin/get-metric-for-build.py --use-apks src/tests/ep1/clock.html nightly.apk

Typically, you want to run Eideticker more than once on a particular test to
get a range of results as tests are not 100% deterministic (partly due to the
way we run tests, partly due to Android itself). You can do this with the
`--num-runs` option. For example:

    ./bin/get-metric-for-build.py --use-apks --num-runs 5 src/tests/ep1/clock.html nightly.apk

If you want to know more about the results (where the numbers are coming from)
you can open them up inside the Eideticker web interface. To open it, execute:

    ./bin/webapp.sh

Then connect to http://localhost:8080. You should see a list of captures, select
the one you're interested in to dive into fine grained detail.

In addition to supporting HDMI capture and analysis, it is also possible to run
the Eideticker harness in a mode that simply "captures" the performance log
of Fennec and outputs results. This has two advantages: first, you don't need
any kind of specialized hardware. Second, it's much faster (since there's no
video encoding/decoding/analysis step). For this you want to pass in
"--no-capture" and "--get-internal-checkerboard-stats", like so:

    ./bin/get-metric-for-build.py --use-apks --no-capture --get-internal-checkerboard-stats src/tests/ep1/taskjs.org/index.html nightly.apk

### Eideticker "dashboard"

Dashboard mode is used to generate a dashboard of eideticker results, like
what you see at http://wrla.ch/eideticker/dashboard. From a toplevel, it
is run from a script called `bin/run-update-dashboard.sh`, which can be called
standalone. This script then it turn calls another script called
`bin/update-dashboard.py` with various arguments corresponding to firefox
version, test to run, etc. This is not yet supported for B2G.

Setting up a new instance of the dashboard has two components: setting up one
or more "clients" (machines that run the tests) and setting up the server
(machines that will serve up the results).

#### Dashboard Server Setup

* Install nginx, ssh, and rsync if not installed already.
* Create an "eideticker" user on the machine (with home directory).
* As the newly-created eideticker user, create a "www" subdirectory.
* Create an nginx configuration where applicable (on Redhat-based systems,
`/etc/nginx/conf.d`). It should look something like this:

```
    server {
            listen 80; #or change this to your public IP address eg 1.1.1.1:80
            server_name eideticker;
            server_name eideticker.wrla.ch;
            access_log /var/log/nginx/eideticker.access_log;
            error_log /var/log/nginx/eideticker.error_log;

            location / {
                    root   /home/eideticker/www;
                    index  index.html index.htm;
                    add_header 'Access-Control-Allow-Origin' '*';
            }
    }
```
Note the Access-Control-Origin header, which allows us to integrate the SPS
profiler to request eideticker resources directly (useful for direct linking
to capture analysis).
* Finally, restart nginx. You will need to configure the clients before you
will see anything on the dashboard.

#### Dashboard Client Setup

Each eideticker client works by creating its own static copy of the dashboard,
then copying the relevant files to the server setup above.

To setup to capture and store results

FIXME: TODO

To setup to synchronize to the eideticker server:

* Generate an ssh key for the client machine, if it doesn't have one already.
* On the server, copy/paste the generated public key into
  `/home/eideticker/.ssh/authorized_keys` (if you're creating this file /
  directory for the first time, be sure that the directory has 700 (rwx only
  for its owner) permissions and the file has 600 (rw only for its owner)
  permissions.
* Set up the sync-dashboard script to synchronize the dashboard data to the
  server on a fixed schedule using cron. Use `crontab -e` to add an entry like
  this:

    0 0 * * * /home/mozauto/src/eideticker/bin/sync-dashboard.sh eideticker.wrla.ch

### Eideticker "tests"

Eideticker tests are simply static HTML with a bit of JavaScript glue to
interface with the harness and some JSON metadata to describe actions that
Eideticker should perform while a capture is ongoing. The simplest example
of a test would probably be the clock demo, which you can find in
`src/tests/ep1/clock/index.html` (you have to run `./bootstrap.sh first to
checkout the ep1 submodule before you can find this file).

Writing your own tests is a matter of adding a subdirectory to `src/tests`,
creating/copying an HTML page of your choice, adding the relevant JavaScript
code to start/stop the test as appropriate, and then making an actions.json
file with whatever actions you want to simulate during the test.

The JavaScript code is encapsulated in the file: `src/tests/js/eideticker.js`.
Much of the code in there is just used by the harness. From the point of view
of an Eideticker test, you really just need to call the "finish" method, which
we use to trigger the end of a capture.

Creating actions for use during a test (and making sure they get called) is
somewhat more involved, but not much. Alongside the various HTML/JS/Image
files that correspond to your page, you also want to create an `actions.json`
file that indicates what you want to happen during the test. Here's an example:

    {
      "default": {
        "LG-P999": [
          ["scroll_down", 4],
          ["sleep", 5]
        ],
        "Galaxy Nexus": [
          ["scroll_down", 7, 3],
          ["sleep", "4"]
        ]
      }
    }

As you can see, this is basically a dictionary of dictionaries. The top-level
is a dictionary of action sets. Normally you just have one, default, but you
can add more in case you want to have multiple tests with different types
of actions (the New York Times test, in `src/tests/ep1/nytimes` would be
an exmaple of this).

The next level down is a dictionary of actions corresponding to different
models of phone (as determined by the `ro.product.model` property of
Android). This is because different types of phones have different input
properties (screen sizes, operating system versions, etc.). The set of
actions is an array of arrays. Each action array corresponds to a single
action performed during a test. As of this writing, there are four possible
actions:

*sleep* [secs]

This action simply sleeps for the corresponding number of seconds (typically
used to allow something to complete in the capture before possibly performing
other actions)

*scroll_down* [number of times] [number of steps]

Triggers a scroll down the specified number of times. Optionally pass the
number of steps parameter to make the action go slower is faster
(lower=faster, default is 10).

*scroll_up* [number of times] [number of steps]

Triggers a scroll up the specified number of times. Optionally pass the
number of steps parameter to make the action go slower is faster
(lower=faster, default is 10).

*double_tap* [x coordinate] [y coordinate]

Triggers two tap events in succession, at the specified x and y coordinates
(this is designed to allow zooming into a web page).

--

Triggering these actions requires posting to the JSON API endpoint on the
eideticker desktop machine running the test. This is typically done as
follows (using jQuery):

    $.post('/api/captures/input', { 'commands': 'default' },
           function() {
               Eideticker.finish();
           });

Obviously you should replace "default" in the above example with the set of
commands that you actually want to run.
