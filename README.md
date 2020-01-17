# Unofficial web app wrapper for Office 365 web apps

A set of web app wrappers, based on QtWebEngine, for the following websites:

- [Outlook](https://outlook.live.com/mail/0/inbox)
- [Word](https://www.office.com/launch/word)
- [Excel](https://www.office.com/launch/excel)
- [PowerPoint](https://www.office.com/launch/powerpoint)
- [OneDrive](https://onedrive.live.com/)
- [OneNote](https://www.onenote.com/notebooks)

These are links to the Office 365 web apps embedded in a minimalist web browser.

This snap is not supported by or endorsed by Microsoft, Inc.

Office 365, OneNote, OneDrive, Outlook, Excel, PowerPoint, and Microsoft are trademarks or registered trademarks of Microsoft, Inc.

## Download

```
snap install webapp-office-x365
```

[![Snap](https://snapcraft.io/static/images/badges/en/snap-store-black.svg)](https://snapcraft.io/webapp-office-x365)

## Notes

Check "Keep me signed in" to avoid having to sign into each web app individually.

## Screenshot

![Screenshot](https://res.cloudinary.com/canonical/image/fetch/f_auto,q_auto,fl_sanitize,w_819,h_614/https://dashboard.snapcraft.io/site_media/appmedia/2019/12/Screenshot_from_2019-12-14_01-27-09.png)

## How This Works

[webapp-office-x365.launcher](https://github.com/sirredbeard/webapp-office-x365/blob/master/webapp-office-x365.launcher) is just a shell script:

```
#!/bin/sh

# Disable the chromium sandbox to work around https://launchpad.net/bugs/1599234.
# Rely on snapd’s security policy instead.
export OXIDE_NO_SANDBOX=1

# Explicitly set APP_ID.
export APP_ID=webapp-office-x365

# Remove problematic environment variable
unset QT_QPA_PLATFORM

exec "$SNAP/bin/desktop-launch" "qmlscene" "$1" "$2" "$SNAP/webapp-office-x365.qml" --scaling

```

The shell script sets some variables and then calls a desktop launcher binary app supplied by a helper kit from the Snapcraft team for running Qt-based GUI desktop apps inside snaps. A [snap](https://snapcraft.io/docs/getting-started) is a Linux app packaged in a container with the app and all its dependencies, including helper apps like this one. A snap is [built using a Snapcraft file](https://snapcraft.io/docs/snapcraft-overview), which is written in [YAML](https://snapcraft.io/docs/snapcraft-format). The snap is then built and pushed to the [Snap store](https://snapcraft.io/store) [using a command-line tool](https://snapcraft.io/docs/releasing-your-app) called snapcraft.

Note that the launcher shell script will run inside our container, so the paths it uses is set relative to the snap using $SNAP and will be interpreted on the fly when the snapped application is run.

A similar GUI helper kit exists for [GTK-based apps](https://github.com/ubuntu/snapcraft-desktop-helpers/tree/master/gtk). Here is the relevant YAML in the [Snapcraft file](https://github.com/sirredbeard/webapp-office-x365/blob/master/snap/snapcraft.yaml) where the helper kit is installed into the snap:

```
parts:
  desktop-qt5:
    source: https://github.com/ubuntu/snapcraft-desktop-helpers.git
    source-subdir: qt
    plugin: make
    make-parameters: ["FLAVOR=qt5"]
```

The helper kit is built directly from the upstream GitHub repository using the [make plugin](https://snapcraft.io/docs/make-plugin).

Back in our original shell script desktop-launch is used to call [qmlscene](https://doc.qt.io/qt-5/qtquick-qmlscene.html). qmlscene is part of the [Qt UI development toolkit](https://www.qt.io/):

`exec "$SNAP/bin/desktop-launch" "qmlscene" "$1" "$2" "$SNAP/webapp-office-x365.qml" --scaling`

qmlscene is used to run mock-ups of QML files, the Qt markup language. There is no separate C++ (or Python) code here you would normally expect to see with a full-fledged Qt application. The logic of the app is embedded in [JavaScript embedded in the QML](https://doc.qt.io/qt-5/qtqml-javascript-expressions.html) and how Snapcraft itself simply works. qmlscene is called with --scaling to enable scaling on HiDPI displays.

The shell script tells qmlscene to parse QML file [webapp-office-x365.qml](https://github.com/sirredbeard/webapp-office-x365/blob/master/webapp-office-x365.qml). The QML draws a basic window, embeds a WebKit engine in it (these Qt dependencies are fulfilled by the staged packages in our [Snapcraft file](https://github.com/sirredbeard/webapp-office-x365/blob/master/snap/snapcraft.yaml)), handles two command-line variables passed to it: the name of the website and it's URL:

```
Window {
  title: Qt.application.arguments[2]
  width: 1000
  height: 600
  visible: true

  WebEngineView {
    id: windowParent
    anchors.fill: parent
    zoomFactor: 1.0

    url: Qt.application.arguments[1]
```

Some JavaScript (currently a bit cumbersome) determines whether to open clicked links in a new window (Microsoft-related links) or in your default OS browser (everything else):

```
 onNewViewRequested: function(request) {
      if (request.requestedUrl.toString().includes('live.com')) {
        var newWindow = windowComponent.createObject(windowParent);
        request.openIn(newWindow.webView);
      }
      else if (request.requestedUrl.toString().includes('microsoft.com')) {
        var newWindow = windowComponent.createObject(windowParent);
        request.openIn(newWindow.webView);
      }

```

The creation of the individual web apps is then handled by Snapcraft itself when the snap is built. For each web app an app is declared in the snap with a corresponding .desktop file (which itself has a corresponding icon) and app, specific command-line arguments, for example `OneDrive` and `https://onedrive.live.com/`.

Taking a look [in our Snapcaft file](https://github.com/sirredbeard/webapp-office-x365/blob/master/snap/snapcraft.yaml), after the Qt GUI desktop helper kit is installed, the contents of our GitHub repo are then dumped into the snap build environment by use of the [dump plugin](https://snapcraft.io/docs/dump-plugin) and necessary Qt dependencies installed by their Debian package names using [stage-packages](https://snapcraft.io/docs/build-and-staging-dependencies):

```
  webapp-office-x365-qml:
    after: [desktop-qt5]
    source: .
    plugin: dump
    stage-packages:
      - qmlscene
      - qml-module-qtquick2
      - qml-module-qtquick-window2
      - qml-module-qtwebengine
      - qml-module-qtquick-dialogs
    organize:
      webapp-office-x365.launcher: bin/webapp-office-x365.launcher
```

[Organize](https://snapcraft.io/docs/snapcraft-parts-metadata) moves our binary into place and is a lot cleaner than a lot of `mkdir` and `mv`. The [gui folder](https://github.com/sirredbeard/webapp-office-x365/tree/master/snap/gui) with its .desktop files and .png icon files will make its way over by virtue of being in the snap folder. Notice how only things declared, moved, or in specific places make it into the snap with our app.

We then 'create' our individual apps for each of the web apps:

```
apps:
  word:
    command: webapp-office-x365.launcher "https://www.office.com/launch/word" "Word"
    desktop: snap/gui/word.desktop
    plugs: &plugs
      - browser-support
      - desktop
      - desktop-legacy
      - home
      - network
      - opengl
      - screen-inhibit-control
      - x11
    environment: &environment
      DISABLE_WAYLAND: 1
```

The plugs are the hooks our snap apps need to interact with the desktop and web, the permissions we need to give the snap. Without [these plugs](https://snapcraft.io/docs/supported-interfaces) the snap is [otherwise fully confined](https://snapcraft.io/docs/interface-management) by default. This allows you to define using useful categories the confines of your app which is then restricted at the kernel level by use of namespaces, cgroups, and App Armor.

Thankfully creating the rest of the apps is not as complicated, we can just * those plugs and other settings. When the app is installed the snapd daemon will move the .desktop files into place on our system, making them visible in the desktop environment. The [.desktop format](https://specifications.freedesktop.org/desktop-entry-spec/latest/) is a FreeDesktop spec.

```
  outlook:
    command: webapp-office-x365.launcher "https://outlook.live.com/mail/0/inbox" "Outlook"
    desktop: snap/gui/outlook.desktop
    plugs: *plugs
    environment: *environment
```

Snapcraft creates our individual apps, all calling [webapp-office-x365.launcher](https://github.com/sirredbeard/webapp-office-x365/blob/master/webapp-office-x365.launcher) with the relevant app name and URL as command line arguments, which then calls qmlscene and renders [unoffocial-webapp-office.qml](https://github.com/sirredbeard/webapp-office-x365/blob/master/webapp-office-x365.qml) giving us a minimalist web browser with our app name as the window title and URL rendered. Some simple JavaScript in the QML handles how to handle opening new Microsoft links in a new window or other links in the external default OS browser. By referencing the respective .desktop file in our [Snapcraft file](https://github.com/sirredbeard/webapp-office-x365/blob/master/snap/snapcraft.yaml) and then the respective .png icon file in our [.desktop file](https://github.com/sirredbeard/webapp-office-x365/blob/master/snap/gui/excel.desktop) the desktop environment icon creation is handled for us.

The snap is built directly on GitHub using GitHub actions, defined [also in YAML](https://github.com/sirredbeard/webapp-office-x365/blob/master/.github/workflows/snapcraft.yml).

The heading defines the workflow name, when the workflow will run (on a push to the repo), and set up our jobs. The primary job is called stable in anticipation of having future release levels, such as edge or beta, depending on the complexity of the app. We will run the action on an Ubuntu 18.04 VM. However, the default Ubuntu 18.04 image on GitHub Actions does not have the various mountpoints and loopback devices needed for Multipass, the hypervisor that powers the snap build, by default. We can get them though by jumping into a Docker container specifically designed for building snaps from the Snapcraft team:

```
name: snapcraft

on: [push]

jobs:
  stable:

    runs-on: ubuntu-18.04
    container:
      image: snapcore/snapcraft
```

Our first step in the job is to checkout our GitHub repository, as we will be dropping these files directly into our snap. 

Then we must do something odd. Even though we are running on an Ubuntu 18.04 VM the [snapcore/snapcraft container](https://hub.docker.com/r/snapcore/snapcraft/) we have to use for the mountpoints is based on Ubuntu 16.04 and a dependency we need, qml-module-qtwebengine, [is not maintained for Ubuntu 16.04](https://packages.ubuntu.com/search?keywords=qml-module-qtwebengine). We need Ubuntu 18.04, the next most recent Ubuntu LTS release. I could copy the mountpoint customizations in the Dockerfile from the upstream snapcore/snapcraft Docker image and create my own Ubuntu 18.04 Docker image. Or I could just replace xenial with bionic in our apt container's sources, update them, and then run distribution upgrade. This seems cumbersome at first but the upstream Snapcraft Docker container and the Xenial to Bionic upgrade path is going to be better maintained by Canonical than I could my own Docker image.

```
    steps:
    - uses: actions/checkout@v1
    - name: Upgrade Snapcraft Docker container to Bionic to get qml-module-qtwebengine
      run: |
        sudo sed -i 's/xenial/bionic/g' /etc/apt/sources.list
        sudo apt update
        sudo apt -y dist-upgrade
```

With our GitHub repo checked out and dumped, our container upgraded to Ubuntu 18.04 (only takes about 2-3 minutes on average), we then build our snap with snapcraft. Snapcraft will parse snapcraft.yaml, spin up a VM using [Multipass](https://multipass.run/), assmble the parts of the app we defined in the order we specified, install all necessary dependencies, and create a containerized app with clearly defined plugs into our Linux environment. We run snapcraft the run task which passes the command to the shell of the GitHub Action VM.

```
    - name: Build .snap
      run: |
        snapcraft    
```

Note: snapcraft is here because we are using the Snapcraft Docker container to build in. Otherwise you would need to install it using `$ snap install snapcraft`.

Next, we have to push our .snap file to the [Snapcraft.io](https://snapcraft.io) store. This is normally done with a key stored in a local snapcraft.cfg file after logging into Snapcraft on the terminal. But GitHub Actions spins up a new VM on each build and while there are ways to persist files between VM instances, let's set that aside for the moment. What we can here though, rather than having our encrypted key floating around on VMs in-between builds, is to login to the Snapcraft Store on an existing Ubuntu device, export the snapcraft.cfg file containing our key, [convert it to base64 string](https://linux.die.net/man/1/base64), and save that text as [a secret in our GitHub Actions](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets) in a way to be can retrieved and decoded on each run. Note the config file and it's key is not saved anywhere in our GitHub repository itself but in the GitHub Actions settings protected with multi-factor authentication.

Note: If your fork this project and want to push to Snapcraft you will need to generate your own config file, copy to your keyboard, and then into GitHub actions:

```
snapcraft login
snapcraft export-login snapcraft.login
base64 snapcraft.login | xsel --clipboard
```

You will also need to reserve a different name from webapp-office-x365 in the Snapcraft file.

We import the base64 string secret as an environmental variable consumable in shell script just like any other environment variable. This is done in the env section of the related job. We then echo that environmental variable back through base64 and save it as a new snapcraft config file in our VM containing our key before pushing to the Store. Snapcraft grabs the key and pushes our snap directly to the Snapcraft Store.

```
    - name: Push .snap to Snapcraft.io
      env:
        SNAPCRAFT_LOGIN_FILE: ${{ secrets.SNAPCRAFT_LOGIN_FILE }}
      run: |
        mkdir .snapcraft
        echo ${SNAPCRAFT_LOGIN_FILE} | base64 --decode --ignore-garbage > .snapcraft/snapcraft.cfg
        snapcraft push --release=stable *.snap
```

* [Snapcraft Documentation](https://snapcraft.io/docs)
* [GitHub Actions](https://help.github.com/en/actions)

### Build Status

![Status](https://github.com/sirredbeard/webapp-office-x365/workflows/snapcraft/badge.svg)

### Snapcraft Store Status

[![Snapcraft](https://snapcraft.io/webapp-office-x365/badge.svg)](https://snapcraft.io/webapp-office-x365)
