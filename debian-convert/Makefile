# debian-convert/Makefile
# This makefile requires GNU Make.

# Version of and paths to pristine Debian Chromium source tree + tarball,
# obtainable by running "apt-get source chromium" on a Debian system
#
VERSION      = 114.0.5735.106
ORIG_SOURCE  = chromium-$(VERSION)
ORIG_TARBALL = chromium_$(VERSION).orig.tar.xz

# Path to local ungoogled-chromium Git repo
# https://github.com/ungoogled-software/ungoogled-chromium.git
#
UNGOOGLED = $(HOME)/src/ungoogled-chromium
UNGOOGLED_PATCHES = $(UNGOOGLED)/patches

# Path to debian-convert/ subdir of ungoogled-software contrib Git repo
# https://github.com/ungoogled-software/contrib.git
# (where this makefile comes from)
#
DEBIAN_CONVERT = .

# Append this to the Debian package version string
#
ADD_VERSION_SUFFIX = test1

# Target distribution (e.g. bookworm, jammy)
#
DISTRIBUTION = jammy

# Uncomment this if you are converting the Debian stable release of
# Chromium. It sometimes requires (slightly) different settings.
#
#DEBIAN_STABLE = 1

# Package maintainer (i.e. you) contact info
#
export DEBFULLNAME = Emmanuel Goldstein
export DEBEMAIL    = cereal@example.com

# List of patches to drop from the patch series to avoid conflicts.
# This list should be as short as possible; run the check-patch-drop-list
# target to verify this.
#
PATCH_DROP_LIST = \
	disable/tests.patch \
	disable/signin.patch \
	disable/third-party-cookies.patch \
	ungoogled/extra/debian/disable/google-api-warning.patch

ifdef DEBIAN_STABLE
PATCH_DROP_LIST += ungoogled/core/debian/disable/unrar.patch
endif

# quilt(1) setup
#
QUILT = quilt --quiltrc /dev/null
export QUILT_DIFF_OPTS = -p
export QUILT_REFRESH_ARGS = -p ab --no-index --no-timestamps

TIME = env time -p

orig_changelog = $(ORIG_SOURCE)/debian/changelog
ifneq ($(wildcard $(orig_changelog)),)
debian_version := $(shell dpkg-parsechangelog --file $(orig_changelog) --show-field Version)
ifndef URGENCY
URGENCY := $(shell dpkg-parsechangelog --file $(orig_changelog) --show-field Urgency)
endif
endif

################

work := $(shell pwd)

ug_source  = ungoogled-chromium-src
ug_tarball = ungoogled-chromium_$(VERSION).orig.tar.xz

# These are used to symbolically specify a dependency on a certain
# variant of Chromium source tree
#
orig_tree_dep = $(ORIG_SOURCE)/debian/source/format
new_tree_dep  = $(ug_source).new/debian/source/format
ug_tree_dep   = $(ug_source)/debian/source/format

# Avoid weird locale-dependent sort orders
#
export LC_COLLATE = C

# A bit of tooling around quilt(1) to apply all patches and automatically
# refresh the ones which need it (unlike "quilt push -a --refresh", which
# refreshes everything unconditionally)
#
# https://savannah.nongnu.org/bugs/?63986
#
quilt_auto_refresh = \
	while :; do \
		(set -x; $(QUILT) push -a -f -q) | tee tmp.quilt.out; \
		! grep -q 'FAILED -- saving rejects' tmp.quilt.out || exit; \
		if tail -n 1 tmp.quilt.out | grep -q 'forced; needs refresh'; then \
			(set -x; $(QUILT) refresh) || exit; \
		else \
			tail -n 1 tmp.quilt.out | grep -q '^Now at patch' || exit; \
			break; \
		fi; \
	done \
	&& rm -f tmp.quilt.out

################

all: stage-6.stamp

# Stage 1
#
# * Create a new ungoogled-chromium source tree
# * Copy in the ungoogled-chromium patches
# * Append the ungoogled-chromium patches to the patch series
#
stage-1.stamp: $(orig_tree_dep) series.add
	test ! -f stage-2.stamp
	rm -rf $(ug_source).new
# Note: Can't speed this up using hardlinks, alas:
# https://savannah.nongnu.org/bugs/?63994
	cp -a $(ORIG_SOURCE) $(ug_source).new
	cp -a $(UNGOOGLED_PATCHES) $(ug_source).new/debian/patches/ungoogled
	rm -f $(ug_source).new/debian/patches/ungoogled/series
	cat series.add >>$(ug_source).new/debian/patches/series
	touch $@

# Stage 2
#
# * Back out any/all applied patches in the source tree
# * Comment out the patches in PATCH_DROP_LIST from the series
# * Apply and refresh all patches (this should consist mainly of removing
#   u-c hunks that refer to files absent from the Debian tree)
#
# If any patches fail to apply due to file changes, then this rule
# will error out.
#
stage-2.stamp: stage-1.stamp $(new_tree_dep)
	test ! -f stage-3.stamp
	cd $(ug_source).new && ($(QUILT) pop -a -f -q || test $$? -eq 2)
	for p in $(PATCH_DROP_LIST); do \
		echo "checking $$p"; \
		grep -Fqx "$$p" $(ug_source).new/debian/patches/series || exit; \
	done
	for p in $(PATCH_DROP_LIST); do \
		perl -pi -e '$$a=$$_; chomp($$a); $$a eq "'"$$p"'" and s/^/#ungoogled#/' \
			$(ug_source).new/debian/patches/series; \
	done
	cd $(ug_source).new && $(quilt_auto_refresh)
	touch $@

stage_3_deps = \
	stage-2.stamp \
	$(new_tree_dep) \
	ungoogled-domain-substitution.sh \
	rules.add

# Stage 3
#
# * Add the domain-substitution script
# * Append ungoogled-chromium section to debian/rules
# * Hook the (un)patch targets into the "build-arch" and "clean" targets
# * Add a line to rename-via-copy the chromedriver binary, as it will
#   be installed in /usr/bin/ and thus cannot keep the original name
# * Update the package changelog
#
stage-3.stamp: $(stage_3_deps)
	test ! -f stage-4.stamp
	cp ungoogled-domain-substitution.sh $(ug_source).new/debian/
	cat rules.add >>$(ug_source).new/debian/rules

	perl -pi \
		-e '/^override_dh_auto_build-arch:/ and s/$$/ patch/;' \
		-e '/^override_dh_auto_clean:/ and s/$$/ unpatch/' \
		$(ug_source).new/debian/rules

	perl -pi -e 'm,^\tcp (out/Release)/chrome_sandbox , and' \
		-e '$$_.="\tcp $$1/chromedriver $$1/ungoogled-chromedriver\n"' \
		$(ug_source).new/debian/rules
	grep -q ungoogled-chromedriver $(ug_source).new/debian/rules

	test -n '$(debian_version)'
	cd $(ug_source).new && debchange \
		--no-conf \
		--package ungoogled-chromium \
		--newversion $(debian_version)$(ADD_VERSION_SUFFIX) \
		$(if $(DISTRIBUTION),--distribution $(DISTRIBUTION)) \
		$(if $(URGENCY),--urgency $(URGENCY)) \
		--no-auto-nmu \
		'New upstream Debian package release.'

	touch $@

stage_4_deps = \
	stage-3.stamp \
	$(new_tree_dep) \
	$(DEBIAN_CONVERT)/replace-name.pl \
	$(DEBIAN_CONVERT)/editcontrol.pl

# Stage 4
#
# * Modify the files under debian/ to refer to "ungoogled-chromium"
#   instead of "chromium" where appropriate. Note that we have to do
#   this selectively, or else things will break!
# * As a special case, due to being a binary installed in /usr/bin/,
#   "chromedriver" needs to become "ungoogled-chromedriver".
#
stage-4.stamp: $(stage_4_deps)
	test ! -f stage-5.stamp
	perl -pi $(DEBIAN_CONVERT)/replace-name.pl \
		$(ug_source).new/debian/*.bug-control \
		$(ug_source).new/debian/*.conffiles \
		$(ug_source).new/debian/*.desktop \
		$(ug_source).new/debian/*.dirs \
		$(ug_source).new/debian/*.install \
		$(ug_source).new/debian/*.links \
		$(ug_source).new/debian/*.lintian-overrides \
		$(ug_source).new/debian/*.manpages \
		$(ug_source).new/debian/*.postinst \
		$(ug_source).new/debian/*.prerm \
		$(ug_source).new/debian/*.xml \
		$(ug_source).new/debian/etc/master_preferences \
		$(ug_source).new/debian/rules \
		$(ug_source).new/debian/scripts/*

	perl -pi -e 's/\b(chromedriver)\b/ungoogled-$$1/' \
		$(ug_source).new/debian/chromium-driver.install

# The Debian package does not use this field (but we will)
	! grep -i XSBC-Original-Maintainer: $(ug_source).new/debian/control

	$(DEBIAN_CONVERT)/editcontrol.pl -i \
		-e '$$field =~ /^(Source|Package|Breaks|Depends|Recommends|Replaces|Suggests)$$/ and s/\b(chromium)\b/ungoogled-$$1/g;' \
		-e 'if ($$field eq "Maintainer") { set_field("XSBC-Original-Maintainer", $$_); $$_="__UGC_MAINTAINER__"; }' \
		-e '$$field eq "Uploaders" and undef $$_;' \
		$(ug_source).new/debian/control
	sed -i 's#__UGC_MAINTAINER__#$(DEBFULLNAME) <$(DEBEMAIL)>#' \
		$(ug_source).new/debian/control

# Check for any double-replacement snafus
	! grep -ri --exclude-dir=patches ungoogled.ungoogled \
		$(ug_source).new/debian

	touch $@

# Stage 5
#
# * Rename all files under debian/ beginning with "chromium" to
#   "ungoogled-chromium", using the file-map script
#
stage-5.stamp: stage-4.stamp $(new_tree_dep) debian-file-map.sh
	test ! -f stage-6.stamp
	op1=: op2='mv -fv' \
	debian_a=$(ug_source).new/debian \
	debian_b=$(ug_source).new/debian \
	sh -e debian-file-map.sh
	touch $@

# Stage 6
#
# * Rename the ungoogled-chromium source tree to reflect completion
# * Build the Debian source package
#
stage-6.stamp: stage-5.stamp $(new_tree_dep) $(ug_tarball)
	rm -rf $(ug_source)
	mv -f $(ug_source).new $(ug_source)
	dpkg-source --abort-on-upstream-changes --build $(ug_source)
	touch $@
	ls -Ll ungoogled-chromium_$(VERSION)*

################

# Addendum to debian/patches/series
#
series.add: $(UNGOOGLED_PATCHES)/series
	(printf '\n#\n# ungoogled-chromium addendum\n#\n\n'; \
	 perl -pe '/^\w/ and s,^,ungoogled/,' $< \
	) >$@

# Addendum to debian/rules
#
rules.add: $(UNGOOGLED)/flags.gn
	(printf '\n#\n# ungoogled-chromium addendum\n#\n\n'; \
	 perl -p -e 's/"/\\"/g; $$p = ($$.==1) ? "defines+=" : " "x9;' \
		-e 's/(.*)/$$p$$1 \\/' \
		$(UNGOOGLED)/flags.gn; \
	 printf '\npatch:\n'; \
	 printf '\ttest -f   ungoogled-domain-substitution.orig.tar \\\n'; \
	 printf '\t|| debian/ungoogled-domain-substitution.sh\n\n'; \
	 printf 'unpatch:\n'; \
	 printf '\ttest ! -f ungoogled-domain-substitution.orig.tar \\\n'; \
	 printf '\t|| tar xf ungoogled-domain-substitution.orig.tar\n'; \
	 printf '\trm -f     ungoogled-domain-substitution.orig.tar\n' \
	) >$@

# Script that maps files from the original debian/ subdirectory to the
# new one, taking renames into account. The script can be run with the
# following variables set:
#
#   op1 = operation to perform on files which DO NOT change name
#   op2 = operation to perform on files which DO change name
#   debian_a = path to first debian/ subdirectory
#   debian_b = path to second debian/ subdirectory
#
debian-file-map.sh: $(orig_tree_dep)
	(cd $(ORIG_SOURCE) && find debian -type f) \
	| grep -v '^debian/patches/' \
	| sort \
	| perl -n \
		-e 'chomp; s/^/\$$/; $$src=$$_;' \
		-e 's,/(chromium),/ungoogled-$$1,;' \
		-e '$$n = $$_ eq $$src ? "1" : "2";' \
		-e '$$src =~ s/^(.debian)/$${1}_a/;' \
		-e 's/^(.debian)/$${1}_b/;' \
		-e 'print "\$$op$$n\t$$src\t$$_\n"' \
	>$@

# ungoogled-chromium uses the exact same .orig.tar file as chromium,
# so just symlink it
#
$(ug_tarball): $(ORIG_TARBALL)
	ln -s $(ORIG_TARBALL) $(ug_tarball)

################

## Domain substitution

# Script to apply the domain substitutions during the Debian package build
# (invoked from the debian/rules "patch" target)
#
ungoogled-domain-substitution.sh: $(UNGOOGLED)/domain_regex.list domain_substitution.list
	rm -f $@.new
	$(UNGOOGLED)/utils/make_domsub_script.py \
		--regex $(UNGOOGLED)/domain_regex.list \
		--files domain_substitution.list \
		--output $@.new
	perl -pi -e 's/^(backup)=/$$1=ungoogled-/' $@.new
	chmod 755 $@.new
	mv -f $@.new $@

# List of files that, after all the patches have been applied, still
# contain Google-related domain names that need to be neutered.
#
domain_substitution.list: $(UNGOOGLED)/domain_regex.list stage-2.stamp $(new_tree_dep)
	$(UNGOOGLED)/devutils/update_lists.py \
		--pruning pruning.list \
		--domain-substitution $@ \
		--domain-regex $(UNGOOGLED)/domain_regex.list \
		--domain-exclude-prefix .pc/ \
		--domain-exclude-prefix debian/ \
		--no-error-unused \
		--tree $(ug_source).new

# Create a tar file containing all the source files modified by the
# domain substitution process.
#
domain_substitution.files.tar: domain_substitution.list stage-2.stamp $(new_tree_dep)
	tar cf - -C $(ug_source).new \
		--verbatim-files-from \
		--files-from=$(work)/domain_substitution.list \
		>$@

# Alternate rule to generate a patch containing the domain substitutions
#
# NOTE: The generated patch is huge (over 30 MB!) and hard to review.
# It is better to use the script if at all possible.
#
ungoogled-domain-substitution.patch: domain_substitution.files.tar domain_substitution.list
	mkdir tmp.src.a tmp.src.b
	cd tmp.src.a && tar xf ../domain_substitution.files.tar
	cd tmp.src.b && tar xf ../domain_substitution.files.tar

	$(UNGOOGLED)/utils/domain_substitution.py \
		apply \
		--regex $(UNGOOGLED)/domain_regex.list \
		--files domain_substitution.list \
		tmp.src.b

	! diff -ru tmp.src.a tmp.src.b >tmp.patch
	rm -rf tmp.src.?
	tail -n +2 tmp.patch \
	| perl -pe 'if(/^(---|\+\+\+) /){s/ tmp\.src\./ /;s/\t.*//}' \
	>$@
	rm -f tmp.patch

################

## Checks

# Check that the Git repo containing $(UNGOOGLED_PATCHES) is checked out
# to a Git tag matching $(VERSION). This ensures that we are using the
# specific revision of the patches intended for the version of Chromium
# being targeted.
#
.PHONY: check-git
check-git: $(UNGOOGLED_PATCHES)/series
	@cd $(UNGOOGLED_PATCHES) && git show -s >/dev/null 2>&1 || exit 0; \
	tag=`git tag --points-at`; \
	test -n "$$tag" || tag=`git tag -l --sort=-creatordate --merged | head -n 1`; \
	echo; \
	echo "target Chromium source version: $(VERSION)"; \
	echo "ungoogled-chromium patches tag: $$tag"; \
	echo; \
	case "$$tag" in $(VERSION)-*) echo '==== $@ OK ===='; exit 0 ;; *) ;; esac; \
	echo 'Please checkout the appropriate Git tag in the ungoogled-chromium repo containing $(UNGOOGLED_PATCHES)/ .' | fmt; \
	exit 1

# Check that every patch listed in PATCH_DROP_LIST needs to be dropped
# in order for the ungoogled-chromium patch series to apply cleanly.
# That is, verify that PATCH_DROP_LIST is minimal, so that we don't drop
# any patches unnecessarily.
#
# (All this does is attempt to apply the combined patch series multiple
# times, each time enabling one of the dropped patches, and verifying
# that the expected conflicts result.)
#
.PHONY: check-patch-drop-list
check-patch-drop-list: stage-1.stamp
	: >tmp.check-pdl.out
	cd $(ug_source).new/debian/patches && cp -p series series.orig
	for p1 in $(PATCH_DROP_LIST); do \
		(cd $(ug_source).new && (set -x; $(QUILT) pop -a -f -q)); \
		echo "==== checking $$p1 drop ===="; \
		(cd $(ug_source).new/debian/patches && cp -p series.orig series); \
		for p2 in $(PATCH_DROP_LIST); do \
			if test "_$$p2" != "_$$p1"; then \
				perl -pi -e '$$a=$$_; chomp($$a); $$a eq "'"$$p2"'" and s/^/#/' \
					$(ug_source).new/debian/patches/series; \
			fi; \
		done; \
		if (cd $(ug_source).new && $(quilt_auto_refresh)); then \
			echo "==== $$p1 drop NOT NEEDED ===="; \
			echo "$$p1: remove from PATCH_DROP_LIST" >>tmp.check-pdl.out; \
		else \
			echo "==== $$p1 drop OK ===="; \
		fi; \
	done
	cd $(ug_source).new/debian/patches && mv -f series.orig series
	@echo; ! grep . tmp.check-pdl.out
	rm -f tmp.check-pdl.out
	@echo '==== $@ OK ===='

check_domsub_deps = \
	domain_substitution.list \
	domain_substitution.files.tar \
	ungoogled-domain-substitution.sh \
	$(UNGOOGLED)/utils/domain_substitution.py

# Check that the modifications made by domain_substitution.py and those
# made by the script created by make_domsub_script.py are identical.
#
.PHONY: check-domsub
check-domsub: $(check_domsub_deps)
	mkdir tmp.src.a tmp.src.b
	cd tmp.src.a && tar xf ../domain_substitution.files.tar
	cd tmp.src.b && tar xf ../domain_substitution.files.tar

	$(TIME) $(UNGOOGLED)/utils/domain_substitution.py \
		apply \
		--regex $(UNGOOGLED)/domain_regex.list \
		--files domain_substitution.list \
		tmp.src.a

	cd tmp.src.b && $(TIME) ../ungoogled-domain-substitution.sh
	rm tmp.src.b/ungoogled-domain-substitution.orig.tar

	diff -ru tmp.src.a tmp.src.b
	rm -rf tmp.src.?
	@echo '==== $@ OK ===='

.PHONY: check
check: check-git check-patch-drop-list check-domsub

################

.PHONY: clean
clean:
	rm -f stage-?.stamp
	rm -f check-domsub.diff
	rm -f debian-file-map.sh
	rm -f domain_substitution.list pruning.list
	rm -f domain_substitution.files.tar
	rm -f rules.add series.add
	rm -f ungoogled-domain-substitution.patch
	rm -f ungoogled-domain-substitution.sh
	rm -rf ungoogled-chromium-src
	rm -rf ungoogled-chromium-src.new
# dpkg-source(1) temporary dir
	rm -rf ungoogled-chromium-src.orig.*
	rm -rf tmp.*
	rm -f ungoogled-chromium_$(VERSION).orig.tar.xz
	rm -f ungoogled-chromium_$(VERSION)-*.debian.tar.xz
	rm -f ungoogled-chromium_$(VERSION)-*.dsc

.DELETE_ON_ERROR:

# EOF
