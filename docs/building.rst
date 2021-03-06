Building
========

XCode
-----
The macOS menu bar application is built using Xcode. To run a development build
of RktMachine, start by opening the RktMachine project file in Xcode.

::

   $ open RktMachine.xcodeproj

Make sure you are not already running RktMachine. Then use the
``Product -> Run`` menu option or the play button in the top left to build and
run the RktMachine source code.


Rebuilding ``root.qcow2``
-------------------------
The ``root.qcow2`` image file is the base file used in new CoreOS VMs to
provide persistent storage in the user home directory and elsewhere. Since the
CoreOS VM initialisation formats the root image if necessary, this image file
only needs to be the right size. In particular, we do not need to create a
filesystem on the image.

In this section, we detail how to create the ``root.qcow2`` image file to
include under ``src/vm/root.qcow2`` in case there is a need to rebuild it, e.g.
to set a new default image size.

SSH to the CoreOS VM and install the ``dev-rktmachine`` container image if not
already available. (See :ref:`dev`.) Then start the container as before.

::

    sudo rkt run \
        --interactive \
        --volume rktmachine,kind=host,source=$(pwd) \
        dev-rktmachine \
        --mount volume=rktmachine,target=/rktmachine

Once in an interactive session on the container, change to the ``/rktmachine``
directory so output will be on the mounted directory and available to the
CoreOS VM after the container is stopped.

Then run the qemu-img command to create the qcow2 disk image file. Finally, exit
the container.

::

    cd /rktmachine
    qemu-img create -f qcow2 root.qcow2 40G
    exit

Once back on the CoreOS VM, the ``root.qcow2`` file will be present in the
current directory. The easiest way to copy the image file to the host macOS
machine is to copy it to the NFS mounted ``/Users`` directory on the CoreOS VM.


Rebuilding ``tools.qcow2``
--------------------------
The ``tools.qcow2`` image file contains binaries needed for VM setup which are
too large to be delivered by Cloud Init. Specifically, it contains:

- buildah binaries for building new container images on the CoreOS VM.
- OCI image and runtime tools.
- skopeo for working with and converting container images on the CoreOS VM.
- docker2aci for converting Docker images to ACI format.
- A statically linked avahid for broadcasting mDNS from the CoreOS VM so it can
  be addressed as ``rktmachine.local`` instead of by IP address.

The executable files are stored in a gzipped tar for compression. The image
file itself cannot be compressed since this causes issues booting with corectl.
The uncompression is not a significant problem since it is only needed for a
one-shot operation on CoreOS VM boot.

As in the previous section, SSH to the CoreOS VM and install the
``dev-rktmachine`` container image if not already available. (See :ref:`dev`.)
Then start the container as before.

::

    sudo rkt run \
        --interactive \
        --volume rktmachine,kind=host,source=$(pwd) \
        dev-rktmachine \
        --mount volume=rktmachine,target=/rktmachine

Once in an interactive session on the container, change to the ``/rktmachine``
directory so output will be on the mounted directory and available to the
CoreOS VM after the container is stopped.

We want to prepare the ``tools.tar.gz`` file first before handling the image
creation. Create a directory to group the archive contents.

::

    cd /rktmachine
    mkdir tools

The buildah_ project is not packaged as a binary so we need to build it
manually. The problem is that CoreOS is a minimal distribution with no support
for installing dynamically linked library files. So to work on the CoreOS VM we
need to build binaries which are predominately statically linked, using only
commonly available dynamic libraries installed on CoreOS already.

For the buildah binary, we need some additional dependencies built as static
libraries. These libraries only need to be installed on the build container.

.. _buildah: https://github.com/projectatomic/buildah

First, libgpgme. Build this as follows:

::

    cd /rktmachine
    wget https://www.gnupg.org/ftp/gcrypt/gpgme/gpgme-1.10.0.tar.bz2
    tar xjvf gpgme-1.10.0.tar.bz2
    cd gpgme-1.10.0
    ./configure --prefix=/usr --enable-static --disable-shared
    make install

And for a static libdevmapper, use:

::

    cd /rktmachine
    git clone git://sourceware.org/git/lvm2.git
    cd lvm2
    ./configure --prefix=/usr --enable-static_link --enable-pkgconfig
    make device-mapper
    make install_device-mapper

Now, start on the buildah compilation by getting the latest buildah sources and
creating a source tree for Go building.

::

    mkdir /rktmachine/go
    export GOPATH=/rktmachine/go

    go get -d github.com/projectatomic/buildah

And build using:

::

    cd $GOPATH/src/github.com/projectatomic/buildah
    make buildah TAGS="apparmor seccomp"
    install -D -m0755 buildah /rktmachine/tools/buildah

The docker2aci_ binary is similarly not available as a binary. The build output
is already a static binary so it can used on the CoreOS VM without difficulty.

.. _docker2aci: https://github.com/appc/docker2aci

In the case of docker2aci the main issue is that the latest version in master
only supports OCI containers which conform to the v1.0.0-rc2 version of the
OCI container specification. The problem here is that buildah produces OCI
containers which match version v1.0.0-rc5 which is incompatible with
v1.0.0.0-rc2.

In order to get a useful docker2aci, we instead use a fork which has updated the
supported OCI container version to one that works with OCI containers produced
by buildah. This fork is maintained at `woofwoofinc/docker2aci`_.

.. _woofwoofinc/docker2aci: https://github.com/woofwoofinc/docker2aci

Change to the ``/rktmachine`` directory and get the forked version of the
docker2aci source code:

::

    cd /rktmachine
    git clone git://github.com/woofwoofinc/docker2aci docker2aci
    cd docker2aci

Run the build script and copy the binary output to the ``tools`` directory.

::

    ./build.sh
    cp bin/docker2aci /rktmachine/tools

Similarly, the `oci-image-tool`_ and `oci-runtime-tool`_ are not available as
binaries but they are also easy to build from source. Again, the build outputs
static binaries so they can be used on the CoreOS VM without difficulty.

.. _oci-image-tool: https://github.com/opencontainers/image-tools
.. _oci-runtime-tool: https://github.com/opencontainers/runtime-tools

Get the OCI sources and create a source tree for Go building.

::

    go get -d github.com/opencontainers/image-tools/cmd/oci-image-tool
    go get -d github.com/opencontainers/runtime-tools/cmd/oci-runtime-tool

And build:

::

    cd $GOPATH/src/github.com/opencontainers/image-tools
    make all
    BINDIR=/rktmachine/tools make install

    cd $GOPATH/src/github.com/opencontainers/runtime-tools
    make all
    BINDIR=/rktmachine/tools make install

The ``BINDIR`` environment setting takes care of installing the binaries into
the mounted ``tools`` image.

Adding skopeo_ is similar again. Compilation from source is required but in
this case static binaries are not the default. They are easily specified in the
build however so it is no difficulty.

.. _skopeo: https://github.com/projectatomic/skopeo

Get the skopeo sources and create a source tree for Go building.

::

    go get -d github.com/projectatomic/skopeo/cmd/skopeo
    cd $GOPATH/src/github.com/projectatomic/skopeo

The skopeo build provides a target for performing a statically linked build. We
use that together with build tags to exclude shared libraries unavailable on
CoreOS.

::

    make binary-local-static BUILDTAGS="containers_image_ostree_stub exclude_graphdriver_devicemapper netgo"

The resulting binary is placed at ``./skopeo``. Copy this to the ``tools``
directory.

::

    cp skopeo /rktmachine/tools

Adding Avahi_ is similarly not provided as a statically linked binary. The
libdaemon0_ dependency also needs to be compiled with ``-fPIC``.

.. _Avahi: http://www.avahi.org
.. _libdaemon0: http://0pointer.de/lennart/projects/libdaemon

Still in the container, change to the ``/rktmachine`` directory.

::

    cd /rktmachine

Then download and extract the libdaemon0 sources.

::

    wget http://0pointer.de/lennart/projects/libdaemon/libdaemon-0.14.tar.gz
    tar xzf libdaemon-0.14.tar.gz
    cd libdaemon-0.14

Configure to build with ``-fPIC`` and without shared libraries. The avahi build
prefers the shared libraries so by not building them we force the compile to use
the static library instead.

::

    ./configure --prefix=/usr --with-pic --disable-shared
    make clean install

Next download the Avahi source.

::

    cd /rktmachine

    wget https://github.com/lathiat/avahi/archive/v0.7.tar.gz
    tar xzf v0.7.tar.gz
    cd avahi-0.7

Use Autoconf/Automake to create a ``./configure`` file. There are a number of
warnings and cautions in the following but the produced binary works okay.

::

    NOCONFIGURE=1 ./autogen.sh

Build avahi with a set of options that turns nearly everything off.

::

    CONFIGURE_OPT="
      --prefix=/rktmachine/install
      --disable-shared
      --disable-glib --disable-gobject
      --disable-qt3 --disable-qt4
      --disable-gtk --disable-gtk3
      --disable-gdbm
      --disable-python --disable-pygtk --disable-python-dbus
      --disable-mono --disable-monodoc
      --disable-doxygen-doc --disable-doxygen-dot --disable-doxygen-html
      --disable-doxygen-xml
      --disable-manpages --disable-xmltoman
      --disable-dbus
      --with-distro=none
      --with-avahi-user=root
      --with-avahi-group=daemon
      --localstatedir=/var
    "

    ./configure ${CONFIGURE_OPT}
    make clean install

All going well, the build artifacts will be in ``/rktmachine/install``. The
only binary we want is ``avahi-daemon`` so copy that to the ``tools``
directory.

::

    cp /rktmachine/install/sbin/avahi-daemon /rktmachine/tools

For the last startup file, create a default container policy file for insertion
at ``/etc/containers/policy.json`` on the CoreOS VM.

::

    $ cat > /rktmachine/tools/policy.json <<EOF
    {
        "default": [
            {
                "type": "insecureAcceptAnything"
            }
        ]
    }
    EOF

Finally build the ``tools.tar.gz`` file.

::

    cd /rktmachine
    GZIP=-9 tar czvf tools.tar.gz tools

Before exiting the container, create a raw image file using QEMU. This is
instead of a qcow2 image file because raw images are easier to mount. Later,
we will convert the raw image to qcow2 format when we are finished creating it.

::

    qemu-img create -f raw tools.raw 32M

Exit the container and format the image file as an ext4 filesystem.

::

    sudo /sbin/mkfs.ext4 -i 8192 -L tools -F tools.raw

Next, mount the ``tools.raw`` image file to the CoreOS VM briefly and copy
``tools.tar.gz`` onto the image.

::

    mkdir tools.mnt
    sudo mount -o loop tools.raw tools.mnt
    sudo cp tools.tar.gz tools.mnt
    sudo umount tools.mnt

Finally restart the container and do the file conversion to create a qcow2
format image from the raw image file.

::

    sudo rkt run \
        --interactive \
        --volume rktmachine,kind=host,source=$(pwd) \
        dev-rktmachine \
        --mount volume=rktmachine,target=/rktmachine \
        --exec /bin/bash

    cd /rktmachine
    qemu-img convert -f raw -O qcow2 tools.raw tools.qcow2

Exit the container and copy the ``tools.qcow2`` image to where it is needed,
typically to the RktMachine repository under ``src/vm/tools.qcow2``. As before,
the easiest way to copy the image file to the host machine is to copy it to
the NFS mounted user directory on the CoreOS VM.

Cleanup the build files on the CoreOS VM.

::

    sudo rm -fr avahi-0.7 docker2aci go gpgme-1.10.0 gpgme-1.10.0.tar.bz2 \
      install libdaemon-0.14 libdaemon-0.14.tar.gz lvm2 tools tools.mnt \
      tools.raw tools.tar.gz v0.7.tar.gz


Rebuilding macOS Corectl Binaries
---------------------------------
The latest versions of the Corectl binaries can be downloaded from the
`Corectl releases`_ for inclusion in the RktMachine application.

.. _Corectl releases: https://github.com/TheNewNormal/corectl/releases

Alternatively the Corectl binaries can be built from source, e.g. to test
changes or for debugging purposes.

Since the Corectl binaries are run on the host macOS machine, it is more
convenient to build on macOS rather than attempting to cross compile in the
development rkt container.

Start by installing the Ocaml and Go compilers as well as the libev compilation
dependency needed to make the qemu-tool binary. (This is unused in RktMachine
but needed for the compile.)

::

    brew install opam go libev

The compilation will require Ocaml version 4.05.0. Check the Ocaml version by
running:

::

    $ ocaml -version
    The OCaml toplevel, version 4.06.0

If you need change the installed version, use the following to unlink the
installed version and to download and install the 4.05.0 version.

::

    brew unlink ocaml
    brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/00f632a7990ac314d63f9cdcb831bea7e8371c61/Formula/ocaml.rb

Verify the Ocaml version is correct.

::

    $ ocaml -version
    The OCaml toplevel, version 4.05.0

Next, clean any previous OPAM installation and set up the Ocaml libraries
needed.

.. CAUTION::
   The following instructions are unsuitable if you normally do Ocaml
   development on your macOS. You are unlikely to appreciate your
   ``~/.opam`` directory being cleared.

::

    rm -fr ~/.opam
    opam init --yes

Create ``~/.ocamlinit`` to avoid compilation problems with topfind.

::

    $ cat > ~/.ocamlinit
    let () =
      try Topdirs.dir_directory (Sys.getenv "OCAML_TOPLEVEL_PATH")
      with Not_found -> ()
    ;;

Continue the installation:

::

    eval `opam config env`
    opam install --yes ocamlfind
    opam install --yes uri
    opam install --yes conf-libev
    opam install --yes qcow-format
    opam install --yes "lwt=3.0.0"
    opam install --yes "io-page=1.6.1"

Do the same for Go. Clean any previous installation and setup for the corectl
build.

.. CAUTION::
   The following instructions are unsuitable if you normally do Go
   development on your macOS. You are unlikely to appreciate your
   ``~/go`` directory being cleared.

::

    export GOPATH=~/go
    rm -fr $GOPATH

Then add the Corectl repository to your Go tree.

::

    git clone https://github.com/TheNewNormal/corectl $GOPATH/src/github.com/TheNewNormal/corectl
    cd $GOPATH/src/github.com/TheNewNormal/corectl

Select the release to build and hack the build file to include a Hyperkit fix.
The second checkout moves to the most recent known good. The cherry pick
includes the latest CoreOS Linux public keys for image signature verification.

::

    git checkout v0.7.18
    git checkout a180a1bff84da47e5f2babd3d1a912f1ab26743c
    git cherry-pick 634cb7bb304f21d5127a5f967838fd8f7702eed2

    sed -i .bk 's/$(GIT) checkout $(HYPERKIT_COMMIT);/$(GIT) checkout $(HYPERKIT_COMMIT); $(GIT) cherry-pick 171c5b060272f821d0d01ff3e9d45639b0073e75;/' Makefile

Finally, perform the build.

::

    make clean
    make tarball

The output binaries are placed in
``~/go/src/github.com/TheNewNormal/corectl/bin``. It is only necessary to
copy ``corectl``, ``corectld``, and ``corectld.runner`` to the RktMachine
repository since the QEMU tool is unused. The binaries should be placed under
``src/bin`` in the RktMachine repository.
