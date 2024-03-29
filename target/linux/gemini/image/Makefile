#
# Copyright (C) 2009-2018 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

# Cook a "WRGG" image, this board is apparently one in the D-Link
# WRGG family and uses the exact same firmware format as other
# D-Link devices.
define Build/dir685-images
	mkwrggimg -i $(IMAGE_KERNEL) \
	-o $@.new \
	-d /dev/mtdblock/1 \
	-s wrgns01_dlwbr_dir685RBWW \
	-v 'N/A' \
	-m dir685 \
	-B 96bb

	mv $@.new $@
endef

# Build D-Link DNS-313 images using the special header tool.
# rootfs.tgz and rd.tgz contains nothing, we only need them
# to satisfy the boot loader on the device. The zImage is
# the only real content.
define Build/dns313-images
	mkdir -p $@.tmp/.boot

	echo "dummy" > $@.tmp/dummyfile

	dns313-header $@.tmp/dummyfile \
		$@.tmp/.boot/rootfs.tgz
	dns313-header $@.tmp/dummyfile \
		$@.tmp/.boot/rd.gz
	dns313-header $(IMAGE_KERNEL) \
		$@.tmp/.boot/zImage

	tar --sort=name --owner=0 --group=0 --numeric-owner -czf $@ \
		-C $@.tmp .boot \
		$(if $(SOURCE_DATE_EPOCH),--mtime="@$(SOURCE_DATE_EPOCH)")

	rm -rf $@.tmp
endef

define Build/wiligear-image
	$(STAGING_DIR_HOST)/bin/mkfwimage2 \
		-m GEOS -f 0x30000000 -z \
		-v $(1).v5.00.SL3512.OpenWrt.00000.000000.000000 \
		-p Kernel:0x020000:0x100000:0:0:$(IMAGE_KERNEL) \
		-p Ramdisk:0x120000:0x500000:0:0:$@ \
		-o $@.new

	mv $@.new $@
endef

# Create the special NAS4220B and Itian Square One SQ201 image
# format with the squashfs and overlay inside the "rd.gz" file.
# We pad it out to 6144K which is the size of the initramfs partition.
#
# The "application" partition is just blank. You can put anything
# there when using OpenWRT. We just use that to create the
# "sysupgrade" firmware image.
define Build/nas4220b-sq201-images
	mkdir -p $@.tmp

	mv $@ $@.tmp/rd.gz
	dd if=/dev/zero of=$@.tmp/hddapp.tgz bs=6144k count=1
	cp $(IMAGE_KERNEL) $@.tmp/zImage
	cp ./ImageInfo-$(1) $@.tmp/ImageInfo

	sed -i -e "s/DATESTR/`date +%Y%m%d $(if $(SOURCE_DATE_EPOCH),--date "@$(SOURCE_DATE_EPOCH)")`/g" $@.tmp/ImageInfo

	(cd $@.tmp; tar --sort=name --owner=0 --group=0 --numeric-owner -czf $@ * \
		$(if $(SOURCE_DATE_EPOCH),--mtime="@$(SOURCE_DATE_EPOCH)"))

	rm -rf $@.tmp
endef

# WBD-111 and WBD-222:
# work around the bootloader's bug with extra nops
# FIXME: is this really needed now that we no longer append the code
# to change the machine ID number? Needs testing on Wiliboard.
define Build/wbd-nops
	mv $@ $@.tmp
	echo -en "\x00\x00\xa0\xe1\x00\x00\xa0\xe1\x00\x00\xa0\xe1\x00\x00\xa0\xe1" > $@
	cat $@.tmp >> $@
	rm -f $@.tmp
endef

# All DTB files are prefixed with "gemini-"
define Device/Default
	PROFILES := Default
	KERNEL_DEPENDS = $$(wildcard $(DTS_DIR)/$$(DEVICE_DTS).dts)
	KERNEL_NAME := zImage
	KERNEL := kernel-bin | append-dtb
	BLOCKSIZE := 128k
endef

# A reasonable set of default packages handling the NAS type
# of devices out of the box (former NAS42x0 IcyBox defaults)
GEMINI_NAS_PACKAGES:=kmod-md-mod kmod-md-linear kmod-md-multipath \
		kmod-md-raid0 kmod-md-raid1 kmod-md-raid10 kmod-md-raid456 \
		kmod-fs-btrfs kmod-fs-cifs kmod-fs-nfs \
		kmod-fs-nfsd kmod-fs-ntfs kmod-fs-reiserfs kmod-fs-vfat \
		kmod-nls-utf8 kmod-usb-storage-extras \
		samba36-server mdadm cfdisk fdisk e2fsprogs badblocks

define Device/dlink_dir-685
	DEVICE_TITLE := D-Link DIR-685 Xtreme N Storage Router
	DEVICE_DTS := gemini-dlink-dir-685
	DEVICE_PACKAGES := $(GEMINI_NAS_PACKAGES) \
			kmod-switch-rtl8366rb swconfig \
			rt2800-pci
	IMAGES := factory.bin
	IMAGE/factory.bin := dir685-images
endef
TARGET_DEVICES += dlink_dir-685

define Device/dlink_dns-313
	DEVICE_TITLE := D-Link DNS-313 1-Bay Network Storage Enclosure
	DEVICE_DTS := gemini-dlink-dns-313
	DEVICE_PACKAGES := $(GEMINI_NAS_PACKAGES)
	IMAGES := factory.bin
	IMAGE/factory.bin := dns313-images
endef
TARGET_DEVICES += dlink_dns-313

define Device/itian-raidsonic
	IMAGES := factory.bin
	IMAGE/factory.bin := append-rootfs | pad-rootfs | pad-to 6144k | \
		nas4220b-sq201-images $(1)
	DEVICE_PACKAGES := $(GEMINI_NAS_PACKAGES)
endef

define Device/itian_sq201
	$(Device/itian-raidsonic)
	DEVICE_TITLE := ITian Square One SQ201
	DEVICE_DTS := gemini-sq201
	DEVICE_PACKAGES += kmod-rt61-pci kmod-usb2-pci
endef
TARGET_DEVICES += itian_sq201

define Device/raidsonic_ib-4220-b
	$(Device/itian-raidsonic)
	DEVICE_TITLE := Raidsonic NAS IB-4220-B
	DEVICE_DTS := gemini-nas4220b
endef
TARGET_DEVICES += raidsonic_ib-4220-b

# The wiliboard images need some changes to be functional and buildable.
#
# The dts would need to use the ecoscentric,redboot-fis-partitions partition
# parser to get the correct partition offsets and size.
#
# The mkfwimage2 call need to be adjusted to reflect the real size of kernel
# and rootfs. It is expected that the OEM firmware adjusts the on flash
# partition table with the values defined in the image header.
define Device/wiliboard_wbd111
	DEVICE_TITLE := Wiliboard WBD-111
	DEVICE_DTS := gemini-wbd111
	KERNEL := kernel-bin | append-dtb | wbd-nops
	IMAGES := factory.bin
	IMAGE/factory.bin := append-rootfs | pad-rootfs | wiligear-image "WILI-S.WILIBOARD"
endef

define Device/wiliboard_wbd222
	DEVICE_TITLE := Wiliboard WBD-222
	DEVICE_DTS := gemini-wbd222
	KERNEL := kernel-bin | append-dtb | wbd-nops
	IMAGES := factory.bin
	IMAGE/factory.bin := append-rootfs | pad-rootfs | wiligear-image "WILI-S.WBD222"
endef

$(eval $(call BuildImage))
