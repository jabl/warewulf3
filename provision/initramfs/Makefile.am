SUBDIRS = capabilities

all: initramfs.cpio

top_srcdir = @top_srcdir@

MACHINE:=$(shell uname -m)

BUSYBOX_VERSION = 1.26.2
BUSYBOX_SOURCE = $(top_srcdir)/3rd_party/GPL/busybox-$(BUSYBOX_VERSION).tar.bz2
BUSYBOX_DIR = busybox-$(BUSYBOX_VERSION)
BUSYBOX_PATCHES = 

E2FSPROGS_VERSION = 1.42.12
E2FSPROGS_SOURCE = $(top_srcdir)/3rd_party/GPL/e2fsprogs-$(E2FSPROGS_VERSION).tar.gz
E2FSPROGS_DIR = e2fsprogs-$(E2FSPROGS_VERSION)
E2FSPROGS_CONFIGARGS = --disable-testio-debug --disable-debugfs --disable-imager --disable-resizer  --disable-defrag --disable-fsck --disable-tls --disable-uuidd --disable-nls --disable-profile

LIBARCHIVE_VERSION = 3.3.1
LIBARCHIVE_SOURCE = $(top_srcdir)/3rd_party/BSD/libarchive-$(LIBARCHIVE_VERSION).tar.gz
LIBARCHIVE_DIR = libarchive-$(LIBARCHIVE_VERSION)
LIBARCHIVE_CONFIGARGS = --enable-bsdtar=static --disable-bsdcpio --enable-xattr --enable-acl --enable-static --disable-shared  --without-zlib --without-bz2lib --without-iconv --without-lzma --without-lzo2 --without-lz4 --without-nettle --without-openssl --without-xml2 --without-expat

PARTED_VERSION = 3.2
PARTED_SOURCE = $(top_srcdir)/3rd_party/GPL/parted-$(PARTED_VERSION).tar.xz
PARTED_DIR = parted-$(PARTED_VERSION)
PARTED_CONFIGARGS = --prefix=/usr --without-readline --disable-static --disable-nls --disable-dynamic-loading --disable-debug

e2fsprogs:
	@ if [ ! -d "_work/$(E2FSPROGS_DIR)" ]; then \
		echo "Preparing to build e2fsprogs" ;\
		mkdir -p _work/ ;\
		tar xzf $(E2FSPROGS_SOURCE) -C _work/ ;\
	fi
	@ if [ ! -f "_work/$(E2FSPROGS_DIR)/misc/mke2fs" ]; then \
		echo "Building e2fsprogs" ;\
		if [ -f "/usr/lib/rpm/config.guess" ]; then \
			cp /usr/lib/rpm/config.guess _work/$(E2FSPROGS_DIR)/config;\
		fi; \
		if [ -f "/usr/lib/rpm/config.sub" ]; then \
			cp /usr/lib/rpm/config.sub _work/$(E2FSPROGS_DIR)/config;\
		fi; \
		(cd _work/$(E2FSPROGS_DIR)/; ./configure $(E2FSPROGS_CONFIGARGS)) ;\
		$(MAKE) -C _work/$(E2FSPROGS_DIR);\
	fi


busybox:
	@ if [ ! -d "_work/$(BUSYBOX_DIR)" ]; then \
		echo "Preparing to build initramfs core" ;\
		mkdir -p _work/ ;\
		tar xjf $(BUSYBOX_SOURCE) -C _work/ ;\
		for i in $(BUSYBOX_PATCHES); do \
			(cd _work/$(BUSYBOX_DIR); patch -p1 < ../../$${i} || exit 1) \
		done ;\
		cp $(srcdir)/busybox.config _work/busybox-$(BUSYBOX_VERSION)/.config ;\
	fi
	@ if [ ! -f "_work/$(BUSYBOX_DIR)/" ]; then \
		echo "Building initramfs core" ;\
		$(MAKE) -C _work/$(BUSYBOX_DIR) busybox ;\
	fi
	@ if [ ! -d "_work/$(BUSYBOX_DIR)/_install" ]; then \
		echo "Installing initramfs core" ;\
		$(MAKE) -C _work/$(BUSYBOX_DIR) install ;\
	fi

libarchive:
	@ if [ ! -d "_work/$(LIBARCHIVE_DIR)" ]; then \
		echo "Preparing to build libarchive" ;\
		mkdir -p _work/ ;\
        echo In directory `pwd` ;\
		tar xzf $(LIBARCHIVE_SOURCE) -C _work/ ;\
	fi
	@ if [ ! -f "_work/$(LIBARCHIVE_DIR)/" ]; then \
		echo "Building libarchive" ;\
		(cd _work/$(LIBARCHIVE_DIR)/; ./configure $(LIBARCHIVE_CONFIGARGS)) ;\
		$(MAKE) -C _work/$(LIBARCHIVE_DIR);\
	fi

parted:
	@ if [ ! -d "_work/$(PARTED_DIR)" ]; then \
		echo "Preparing to build parted" ;\
		mkdir -p _work/ ;\
        echo In directory `pwd` ;\
		tar Jxf $(PARTED_SOURCE) -C _work/ ;\
	fi
	@ if [ ! -f "_work/$(PARTED_DIR)/" ]; then \
		echo "Building parted" ;\
		(cd _work/$(PARTED_DIR)/; ./configure $(PARTED_CONFIGARGS)) ;\
		$(MAKE) -C _work/$(PARTED_DIR);\
	fi

rootfs: busybox e2fsprogs libarchive parted
	rm -rf rootfs
	mkdir rootfs
	mkdir rootfs/bin
	mkdir rootfs/sbin
	mkdir rootfs/etc
	mkdir rootfs/proc
	mkdir rootfs/var
	mkdir rootfs/tmp
	mkdir rootfs/usr
	mkdir rootfs/lib
	mkdir rootfs/lib64
	chmod -R u+w rootfs/
	cp -a _work/$(BUSYBOX_DIR)/_install/* rootfs/
	cp _work/$(E2FSPROGS_DIR)/misc/mke2fs rootfs/sbin/mkfs.ext4
	ln -s mkfs.ext4 rootfs/sbin/mkfs.ext3
	cp -a _work/$(LIBARCHIVE_DIR)/bsdtar rootfs/bin/bsdtar
	$(MAKE) -C _work/$(PARTED_DIR)/ DESTDIR=`pwd`/rootfs install
	cp -L --parents /lib*/ld-linux* rootfs/
	find rootfs -type f -perm -o+x -print | grep -v ld-linux | xargs ldd | grep "=>" | awk '{print $$3}' | grep "^/" | sort | uniq | while read i; do cp -L --parents $$i rootfs/ && chmod 755 rootfs/$$i; done
	rm -f rootfs/linuxrc rootfs/lib64/*.la rootfs/lib/*.la rootfs/usr/lib64/*.la rootfs/usr/lib/*.la
	rm -rf rootfs/usr/share rootfs/usr/include rootfs/usr/lib/pkgconfig
	find -type d \! -perm -u=w -exec chmod u+w {} \;
	find rootfs -type f -perm -o+x -print | xargs strip -g

	install -m 755 $(srcdir)/init rootfs/init
	install -m 755 $(srcdir)/functions rootfs/etc/functions
	install -m 755 $(srcdir)/dhcp-script rootfs/sbin/
	install -m 755 $(srcdir)/detect rootfs/sbin/
	install -m 755 $(srcdir)/transporthandler rootfs/sbin/
	install -m 755 $(srcdir)/provisionhandler rootfs/sbin/
	install -m 755 $(srcdir)/wwlogger rootfs/sbin/
	ln -s transporthandler rootfs/sbin/wwgetvnfs
	ln -s transporthandler rootfs/sbin/wwgetscript
	ln -s transporthandler rootfs/sbin/wwgetnodeconfig
	ln -s transporthandler rootfs/sbin/wwgetfiles

initramfs.cpio: rootfs
	install -m 644 $(srcdir)/devs.cpio initramfs.cpio
	cd rootfs/; find . | cpio -o -H newc -A -F ../initramfs.cpio

install-data-local: initramfs.cpio
	install -d -m 755 $(DESTDIR)/$(WAREWULF_STATEDIR)/warewulf/initramfs/
	install -d -m 755 $(DESTDIR)/$(WAREWULF_STATEDIR)/warewulf/initramfs/$(MACHINE)
	install -m 644 initramfs.cpio $(DESTDIR)/$(WAREWULF_STATEDIR)/warewulf/initramfs/$(MACHINE)/base

uninstall-local:
	rm -rf $(DESTDIR)/$(WAREWULF_STATEDIR)/warewulf/initramfs/$(MACHINE)/base

clean-local:
	rm -rf _work rootfs initramfs.cpio



EXTRA_DIST = busybox.config init devs.cpio functions dhcp-script detect transporthandler provisionhandler wwlogger

MAINTAINERCLEANFILES = Makefile.in
