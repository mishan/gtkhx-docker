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
#   libnice-gstreamer1 / libsoup3 / gstreamer1-* .. voice (WebRTC) stack
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
        libasan libubsan clang-tools-extra \
        libnice-gstreamer1 \
        libsoup3-devel \
        gstreamer1-devel \
        gstreamer1-plugins-base-devel \
        gstreamer1-plugins-bad-free-devel \
        gstreamer1-plugins-bad-free-extras \
        gstreamer1-plugins-good \
    && dnf clean all \
    && rm -rf /var/cache/dnf

# Sanity: print the toolchain versions at build time so the publish log
# records exactly what a given tag contains. (The GtkHx CI still runs its
# own MSRV assertion against the tree; this is just a build-time record.)
RUN gcc --version | head -1 \
    && meson --version \
    && rustc --version \
    && cargo --version \
    && mold --version | head -1
