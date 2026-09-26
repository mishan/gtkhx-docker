# GtkHx CI base image.
#
# Fedora + the full GtkHx build/test dependency set, pre-installed so CI
# runs skip the per-run `dnf install` (which otherwise costs a couple of
# minutes downloading + installing on every job).
#
# The GtkHx repo's CI builds the tree TWICE — voice-disabled (no
# GStreamer) then voice-enabled — so historically it installed the base
# deps first and the GStreamer stack second. Here we bake BOTH sets in;
# a pre-built image can't lazily split them, and having GStreamer present
# for the voice-disabled build is harmless (that build just doesn't use
# it). The package list mirrors the two `dnf -y install` blocks in the
# GtkHx repo's .github/workflows/tests.yml — keep them in sync.
#
# Fedora is pinned to a major (43) so a surprise-breaking Fedora release
# doesn't silently change the toolchain under CI. 43 specifically because
# its glycin-loaders ship the 2+ loader protocol the `glycin` 3.x crate
# (and the GNOME 49 Flatpak runtime GtkHx targets) need.
#
# Build:   docker build -t gtkhx-ci-base images/ci-base
# Publish: pushed to ghcr.io/mishan/gtkhx-ci-base by
#          .github/workflows/publish-ci-base.yml
# Consume: see docs/consuming-in-gtkhx-ci.md

FROM fedora:43

# One layer, cache cleaned, and installed with tsflags=nodocs (skip man
# pages + /usr/share/doc) so the published image stays lean — it's pulled
# onto every CI runner, where it shares a ~14 GB disk with the cargo target
# dir and the test containers. (The Android/.NET/GHC/CodeQL cruft that used
# to fill that disk lives on the ubuntu-latest runner host, not in this
# image; the GtkHx repo's tests.yml reclaims it before the build.)
#
#   meson/ninja/gcc/pkgconf ....... build system + compiler
#   gtk4 / libadwaita / glib / gir  toolkit
#   alsa-lib ...................... chat alert playback (cpal backend behind
#                                   the hxsound Rust crate; replaced gsound)
#   gettext ....................... i18n build
#   desktop-file-utils / appstream  validate-desktop / validate-metainfo tests
#   libpanel ...................... UI dep
#   libseccomp / lcms2 / fontconfig misc deps (glycin sandbox, colour, fonts)
#   glycin-loaders ................ hx-image-decode decode tests (2+ loaders)
#   git / rust / cargo ......... VCS + Rust workspace
#   rustfmt / clippy ............. lint (cargo fmt --check, cargo clippy)
#   mold .......................... fast linker (GtkHx CI links with `mold -run`)
#   xorg-x11-server-Xvfb .......... virtual display for the widget tests; see below
#   xorg-x11-xauth ................ xvfb-run shells out to it for the cookie
#   libnice-gstreamer1 / libsoup3 / gstreamer1-* .. voice (WebRTC) stack
#   pipewire / wireplumber / pipewire-pulseaudio / pipewire-gstreamer /
#   pipewire-utils / dbus-daemon / dbus-tools .. the private audio session
#                                   tests run in; see below
#
# On Xvfb: GTK 4 has no headless backend, and gtk4-rs asserts an initialised
# GTK inside the generated class_init, so any Rust test that constructs a
# widget needs a real GdkDisplay. Without one those tests don't fail — they
# return early and report success, which is how a reference-cycle leak in the
# chat view stayed invisible for the whole life of the widget while CI showed
# green. GtkHx's tests.yml runs the Rust workspace suite under `xvfb-run -a`
# with GDK_BACKEND=x11; this is the package that makes that work.
#
# `xvfb-run` lives in xorg-x11-server-Xvfb alongside the server binary itself
# (it moved there from xorg-x11-server-utils some releases back). It is a shell
# wrapper that generates an auth cookie via xauth, hence the second package —
# it is not pulled in as a dependency of the first.
#
# On PipeWire: GtkHx's tests run under its tools/isolated-run.sh, which gives
# each run its own D-Bus session bus and its own PipeWire + WirePlumber (plus
# pipewire-pulse, for GStreamer's pulse elements) with a null sink and source,
# so the voice tests have a real audio server to play into without one on the
# host. pipewire-utils is for wpctl, which the wrapper waits on; dbus-tools
# carries dbus-run-session. Nothing here starts at build time: the wrapper
# starts and stops the whole session per run.
RUN dnf -y install --setopt=tsflags=nodocs \
        meson ninja-build pkgconf-pkg-config gcc \
        gtk4-devel libadwaita-devel \
        glib2-devel gobject-introspection-devel \
        alsa-lib-devel \
        gettext-devel \
        desktop-file-utils appstream \
        libpanel-devel \
        libseccomp-devel lcms2-devel fontconfig-devel \
        glycin-loaders \
        git rust cargo rustfmt clippy \
        mold \
        xorg-x11-server-Xvfb xorg-x11-xauth \
        libasan libubsan clang-tools-extra \
        libnice-gstreamer1 \
        libsoup3-devel \
        gstreamer1-devel \
        gstreamer1-plugins-base-devel \
        gstreamer1-plugins-bad-free-devel \
        gstreamer1-plugins-bad-free-extras \
        gstreamer1-plugins-good \
        pipewire wireplumber pipewire-pulseaudio pipewire-gstreamer \
        pipewire-utils dbus-daemon dbus-tools \
    && dnf clean all \
    && rm -rf /var/cache/dnf

# Sanity: print the toolchain versions at build time so the publish log
# records exactly what a given tag contains. (The GtkHx CI still runs its
# own MSRV assertion against the tree; this is just a build-time record.)
RUN gcc --version | head -1 \
    && meson --version \
    && rustc --version \
    && cargo --version \
    && mold --version | head -1 \
    && pipewire --version | tail -1 \
    && wireplumber --version | tail -1 \
    && command -v pipewire-pulse wpctl dbus-run-session

# Prove the virtual display actually works, rather than just that the binaries
# landed. This exercises the whole path GtkHx CI depends on — xvfb-run picking
# a free server number, minting a cookie through xauth, starting Xvfb, running
# a command against it — so a package split, a rename, or a missing xauth fails
# here, at publish time, with an obvious cause. The alternative is a GtkHx CI
# run failing somewhere inside `cargo test` with a GTK assertion.
#
# (No version line for Xvfb above: it has no `-version` flag, and the X server
# it reports is `xorg-x11-server-common`'s anyway.)
RUN xvfb-run -a sh -c 'test -n "$DISPLAY"'
