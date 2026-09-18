# SPDX-License-Identifier: GPL-3.0-or-later
#
# Copyright (c) 2018-2025 Nick Peng (pymumu@gmail.com)

include $(TOPDIR)/rules.mk

PKG_NAME:=smartdns
PKG_VERSION:=48.4
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://codeload.github.com/pymumu/smartdns/tar.gz/Release$(PKG_VERSION)?
PKG_HASH:=b07abba99e921f262ba47588e12f5b36dd849f79c0145542440eb6b7eed80553
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-Release$(PKG_VERSION)

PKG_MAINTAINER:=Nick Peng <pymumu@gmail.com>
PKG_LICENSE:=GPL-3.0-or-later
PKG_LICENSE_FILES:=LICENSE
PKG_CPE_ID:=cpe:/a:pymumu:smartdns

PKG_CONFIG_DEPENDS:=CONFIG_PACKAGE_smartdns-ui

# smartdns-ui is a Rust plugin (plugin/smartdns-ui) plus a Next.js dashboard
# (pymumu/smartdns-webui, built with npm), so the UI variant needs, on the build
# host:
#   - cargo/rustc with the aarch64-unknown-linux-musl std installed, e.g.
#     `rustup target add aarch64-unknown-linux-musl` (>= 1.87: the plugin's
#     dependency tree, e.g. time and yoke-derive, no longer builds with the 1.85
#     that this branch's lang/rust provides);
#   - node/npm for the frontend;
#   - libclang, for bindgen (which cargo pulls in as a build-dependency).
# Upstream master instead depends on PACKAGE_smartdns-ui:node/host and
# PACKAGE_smartdns-ui:rust-bindgen/host, neither of which exists in this branch,
# and it still relies on a rustc newer than this branch's lang/rust.
PKG_BUILD_PARALLEL:=1

RUST_PKG_FEATURES:=build-release
RUST_PKG_LOCKED:=0

include $(INCLUDE_DIR)/package.mk
# Only for the RUSTC_*/CARGO_PKG_* variables (cross target, linker, profile); the
# plugin is built with the build host's cargo, not with lang/rust.
include ../../lang/rust/rust-package.mk

# cargo's own -j; PKG_JOBS is empty here, and upstream master's `-j1` makes a
# ~300-crate cross build needlessly slow.
CARGO_JOB_ARGS:=$(if $(filter --jobserver%,$(PKG_JOBS)),,-j$(if $(PKG_JOBS),$(PKG_JOBS),$(shell nproc)))

MAKE_PATH:=src
MAKE_VARS+= VER=$(PKG_VERSION) WITH_ZLIB=0

# bindgen needs a real sysroot or it parses size_t/ssize_t as 32-bit and aborts
# with "The size of `ssize_t` (4) does not match the target pointer size (8)".
# rules.mk only fills in TOOLCHAIN_ROOT_DIR for external toolchains, so fall back
# to TOOLCHAIN_DIR here.
SMARTDNS_UI_SYSROOT:=$(if $(TOOLCHAIN_ROOT_DIR),$(TOOLCHAIN_ROOT_DIR),$(TOOLCHAIN_DIR))

SMARTDNS_WEBUI_VERSION:=1.0.0
SMARTDNS_WEBUI_SOURCE_VERSION:=1c06fee693434ec7c71f06010d3f87416baba9f0
SMARTDNS_WEBUI_FILE:=smartdns-webui-$(SMARTDNS_WEBUI_VERSION)-$(call version_abbrev,$(SMARTDNS_WEBUI_SOURCE_VERSION)).tar.gz

define Package/smartdns/Default
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=IP Addresses and Names
  TITLE:=smartdns
  URL:=https://www.github.com/pymumu/smartdns/
endef

define Package/smartdns
  $(call Package/smartdns/Default)
  TITLE+= server
  DEPENDS:=+i386:libatomic +libpthread +libopenssl
endef

define Package/smartdns/description
SmartDNS is a local DNS server which accepts DNS query requests from local network clients,
gets DNS query results from multiple upstream DNS servers concurrently, and returns the fastest IP to clients.
Unlike dnsmasq's all-servers, smartdns returns the fastest IP, and encrypt DNS queries with DoT or DoH.
endef

define Package/smartdns-ui
  $(call Package/smartdns/Default)
  TITLE+= dashboard
  DEPENDS:=+smartdns $(RUST_ARCH_DEPENDS) @!i386
endef

define Package/smartdns-ui/description
The smartdns dashboard (WebUI): a Rust plugin for the smartdns server plus a
Next.js frontend, installed to /usr/lib/smartdns_ui.so and
/usr/share/smartdns/wwwroot. Enable it with 'uci set smartdns.@smartdns[0].ui=1'
(default port 6080), which this package does at first boot.
endef

define Package/smartdns/conffiles
/etc/config/smartdns
/etc/smartdns/address.conf
/etc/smartdns/blacklist-ip.conf
/etc/smartdns/custom.conf
/etc/smartdns/domain-block.list
/etc/smartdns/domain-forwarding.list
endef

define Package/smartdns-ui/conffiles
/etc/config/smartdns
endef

# No git source version handling here (this tree does not support SOURCE_DATE in
# Download/ blocks): pin the frontend by commit and hash the codeload tarball.
define Download/smartdns-webui
  FILE:=$(SMARTDNS_WEBUI_FILE)
  URL:=https://codeload.github.com/pymumu/smartdns-webui/tar.gz/$(SMARTDNS_WEBUI_SOURCE_VERSION)?
  HASH:=7a698dd4826750869429e83d91ce99a4e474d0809588a7f4124ef86875be9b1f
  SUBDIR:=smartdns-webui
endef

define Build/Prepare
	$(call Build/Prepare/Default)

ifneq ($(CONFIG_PACKAGE_smartdns-ui),)
	$(eval $(call Download,smartdns-webui))
	$(eval $(Download/smartdns-webui))
	rm -rf $(PKG_BUILD_DIR)/smartdns-webui
	$(INSTALL_DIR) $(PKG_BUILD_DIR)/smartdns-webui
	$(TAR) -C $(PKG_BUILD_DIR)/smartdns-webui -xf $(DL_DIR)/$(SMARTDNS_WEBUI_FILE) --strip-components=1
endif
endef

define Build/Compile
	$(call Build/Compile/Default)

ifneq ($(CONFIG_PACKAGE_smartdns-ui),)
	( \
		command -v npm >/dev/null || { echo "smartdns-ui: npm not found on the build host"; exit 1; } ; \
		command -v cargo >/dev/null || { echo "smartdns-ui: cargo not found on the build host"; exit 1; } ; \
		pushd $(PKG_BUILD_DIR)/smartdns-webui ; \
		npm install --no-audit --no-fund ; \
		npm run build ; \
		popd ; \
		pushd $(PKG_BUILD_DIR)/plugin/smartdns-ui ; \
		$(CARGO_PKG_CONFIG_VARS) \
		MAKEFLAGS="$(PKG_JOBS)" \
		WITH_ZLIB=0 \
		RUSTFLAGS="$(CARGO_RUSTFLAGS)" \
		TARGET_CFLAGS="$(filter-out -O%,$(TARGET_CFLAGS)) $(RUSTC_CFLAGS)" \
		BINDGEN_EXTRA_CLANG_ARGS="--sysroot=$(SMARTDNS_UI_SYSROOT)" \
		cargo build -v --profile $(CARGO_PKG_PROFILE) \
		$(if $(strip $(RUST_PKG_FEATURES)),--features "$(strip $(RUST_PKG_FEATURES))") \
		$(CARGO_JOB_ARGS) \
		$(CARGO_PKG_ARGS) ; \
		popd ; \
	)
endif
endef

define Package/smartdns/install
	$(INSTALL_DIR) $(1)/usr/sbin $(1)/etc/config $(1)/etc/init.d
	$(INSTALL_DIR) $(1)/etc/smartdns $(1)/etc/smartdns/domain-set $(1)/etc/smartdns/conf.d/
	$(INSTALL_DIR) $(1)/etc/smartdns/ip-set $(1)/etc/smartdns/download
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/src/smartdns $(1)/usr/sbin/smartdns
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/package/openwrt/files/etc/init.d/smartdns $(1)/etc/init.d/smartdns
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/address.conf $(1)/etc/smartdns/address.conf
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/blacklist-ip.conf $(1)/etc/smartdns/blacklist-ip.conf
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/custom.conf $(1)/etc/smartdns/custom.conf
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/domain-block.list $(1)/etc/smartdns/domain-block.list
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/domain-forwarding.list $(1)/etc/smartdns/domain-forwarding.list
	$(INSTALL_CONF) $(PKG_BUILD_DIR)/package/openwrt/files/etc/config/smartdns $(1)/etc/config/smartdns
endef

define Package/smartdns-ui/install
	$(INSTALL_DIR) $(1)/usr/lib
	$(INSTALL_DIR) $(1)/usr/share/smartdns/wwwroot
	$(INSTALL_DIR) $(1)/etc/uci-defaults
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/plugin/smartdns-ui/target/$(RUSTC_TARGET_ARCH)/$(CARGO_PKG_PROFILE)/libsmartdns_ui.so $(1)/usr/lib/smartdns_ui.so
	$(CP) $(PKG_BUILD_DIR)/smartdns-webui/out/* $(1)/usr/share/smartdns/wwwroot/
	$(INSTALL_BIN) ./files/smartdns-ui.defaults $(1)/etc/uci-defaults/95-smartdns-ui
endef

$(eval $(call BuildPackage,smartdns))
$(eval $(call BuildPackage,smartdns-ui))