HSHLOCALCFG ?= Makefile.config

-include $(HSHLOCALCFG)

HSHROOT ?= src
HSHPOOLDIR ?= ""
HSHTPLDIR ?= tpl
HSHPACKETS ?= $(shell find $(HSHROOT) -maxdepth 1 -mindepth 1 -type d -exec basename '{}' \;)
HSHMODIFIED ?= 0
HSHCOLOR ?= 1

ifeq ($(HSHPOOLDIR), "")
HSHDEST ?= $(HSHROOT)
else
HSHDEST ?= $(HSHPOOLDIR)
endif


ifeq ($(HSHCOLOR), 1)
HSHCOLORPKT ?= \033[0;32m
HSHCOLORMSG ?= \033[0;33m
HSHCOLORRST = \033[00m
endif

SRC := $(HSHPACKETS:%=$(HSHROOT)/%)
PWD := $(shell pwd)

.PHONY: all opts
.PHONY: all %/discard %/pack %/set-version %/set-branch %/check-modified %/divert %/divert-files
.PHONY: status %/status
.PHONY: diff %/diff

.PRECIOUS: $(SRC) $(SRC:%=%/.git)
.PRECIOUS: $(SRC:%=%/src/DEBIAN) $(SRC:%=%/src/DEBIAN/control)
.PRECIOUS: $(SRC:%=%/src/DEBIAN/postrm) $(SRC:%=%/src/DEBIAN/preinst)
.PRECIOUS: $(HSHPOOLDIR) $(HSHPOOLDIR)/Packages $(HSHPOOLDIR)/Packages.bz2

all: $(HSHPACKETS:%=%/discard)
	@if [ -n $(HSHPOOLDIR) ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)updating packages file$(HSHCOLORRST)'; \
		dpkg-scanpackages $(HSHPOOLDIR) > $(HSHPOOLDIR)/Packages; \
		bzip2 -kf $(HSHPOOLDIR)/Packages; \
	fi

status: $(HSHPACKETS:%=%/status)

diff: $(HSHPACKETS:%=%/diff)

opts:
	@echo "pooldir: $(HSHPOOLDIR)"; \
	echo "dir: $(HSHROOT)"; \
	echo "packets: $(HSHPACKETS)"

%/discard: %/pack
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)discarding changes$(HSHCOLORRST)'; \
	git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout -- src/DEBIAN/control; \
	dd=$(HSHROOT)/$*/src/DEBIAN; \
	[ -e $$dd/preinst ] && rm $$dd/preinst; \
	[ -e $$dd/postrm ] && rm $$dd/postrm; \
	[ -e $(HSHROOT)/$*/divert.files ] && rm $(HSHROOT)/$*/divert.files; \
	true

%/pack: %/set-version $(HSHPOOLDIR) $(HSHDEST)
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)packing$(HSHCOLORRST)'; \
	rm -f $(HSHDEST)/$*_*.deb; \
	fakeroot sh -c "chown -R root:root $(HSHROOT)/$*/src;dpkg-deb -b $(HSHROOT)/$*/src $(HSHDEST)";

$(HSHPOOLDIR):
	@[ -n $(HSHPOOLDIR) ] \
	&& echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating dir $(HSHPOOLDIR)$(HSHCOLORRST)' \
	&& mkdir -p $(HSHPOOLDIR); \
	true

%/set-version: %/divert
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting version$(HSHCOLORRST)'; \
	sed -i "s/^Version: {{git}}$$/Version: `git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git describe --tags \
	| sed 's/^[^0-9]*//'`/" $(HSHROOT)/$*/src/DEBIAN/control;

%/divert: %/divert-files
	@git="git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git"; \
	if [ -e "$(HSHROOT)/$*/divert.files" ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN/preinst|postrm$(HSHCOLORRST)'; \
		for df in preinst postrm; do \
			of="$(HSHROOT)/$*/src/DEBIAN/$$df"; \
			echo "#!/bin/sh\nset -e\n" > $$of; \
			if [ "$$df" = "preinst" ]; then \
				for f in `cat $(HSHROOT)/$*/divert.files`; do \
					echo "mkdir -p /var/backups`dirname $$f`\n" >> $$of; \
					echo "if [ \"\$$1\" = \"install\" ]; then" >> $$of;  \
					echo "\tdpkg-divert --package $* --rename \\" >> $$of; \
					echo "\t\t--divert /var/backups$$f $$f" >> $$of; \
					echo "fi" >> $$of; \
				done; \
			elif [ "$$df" = "postrm" ]; then \
				for f in `cat $(HSHROOT)/$*/divert.files`; do \
					echo "if [ remove = \"\$$1\"" \
						" -o abort-install = \"\$$1\"" \
						" -o disappear = \"\$$1\" ]; then" >> $$of; \
					echo "\tdpkg-divert --package $* --rename --remove $$f" >> $$of; \
					echo "fi" >> $$of; \
				done; \
			fi ;\
			echo "\nexit 0" >> $$of; \
			chmod a+x $$of; \
		done; \
	else \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)removing DEBIAN/preinst|postrm$(HSHCOLORRST)'; \
		for df in preinst postrm; do \
			of="src/DEBIAN/$$df"; \
			[ -e $$of ] && rm $(HSHROOT)/$*/$$of; \
		done; \
	fi; \
	true

%/divert-files: %/check-modified
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diverting files$(HSHCOLORRST)'; \
	git="git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git"; \
	df=$(HSHROOT)/$*/divert ;\
	[ -e $$df.files ] && rm $$df.files;\
	[ -e $$df ] && cat $$df > $$df.files ;\
	files=`find $(HSHROOT)/$*/src -type f | grep -v DEBIAN | sed 's/$(HSHROOT)\/$*\/src//'`; \
	for f in $$files; do \
		if [ -e "$$f" ]; then \
			if [ `dpkg -S $$f 2>/dev/null | egrep -v '^$*' -c` -gt 0 ]; then\
				echo "$$f" >> $$df.files; \
			fi; \
		fi; \
	done;

%/check-modified: %/set-branch
	@if [ $(HSHMODIFIED) -eq 0 ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)cheking repository modification$(HSHCOLORRST)'; \
		if [ ! $$(git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git status --porcelain | wc -l) -eq 0 ]; then \
			echo "$*: modified"; >&2 \
			exit 2; \
		fi; \
	fi

%/set-branch: $(HSHROOT)/%/.git
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting branch to "debian"$(HSHCOLORRST)'; \
	if [ $$(git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git branch | wc -l) -eq 0 ]; then \
		echo "$*: not initialized" >&2; \
		exit 1; \
	fi; \
	git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout debian 2>&1; \
	[ $$? -eq 1 ] && git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout -b debian 2>&1 && true; \
	true

$(HSHROOT)/%/.git: $(HSHROOT)/%/src/DEBIAN/control
	@if [ ! -d $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)git init && first commit$(HSHCOLORRST)'; \
		cd $(HSHROOT)/$* \
		&& git init \
		&& git add * \
		&& git ci -m 'initial commit' \
		&& git tag 0.0.0 \
		&& cd $(PWD); \
	fi

$(HSHROOT)/%/src/DEBIAN/control: $(HSHROOT)/%/src/DEBIAN
	@if [ ! -e $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting DEBIAN/control$(HSHCOLORRST)'; \
		sed "s/Package: {{name}}/Package: $*/" $(HSHTPLDIR)/DEBIAN/control \
		| sed "s/Description: {{description}}/Description: $*/" \
		| sed "s/Maintainer: {{maintainer}}/Maintainer: `git config --get user.name` <`git config --get user.email`>/" > $@; \
	fi

$(HSHROOT)/%/src/DEBIAN:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN directory$(HSHCOLORRST)'; \
	mkdir -p $@

%/status:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)status$(HSHCOLORRST)'; \
	msg=`git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git status --porcelain`; \
	[ -n "$$msg" ] && printf "$*:\n$$msg\n"; \
	true

%/diff:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diff$(HSHCOLORRST)'; \
	files=`find $(HSHROOT)/$*/src -type f | grep -v DEBIAN | sed "s/$(HSHROOT)\/$*\/src//"`; \
	diff=""; \
	for of in $$files; do \
		msg=""; \
		if [ -e "$$of" ]; then \
			diff $$of $(HSHROOT)/$*/src/$$of > /dev/null; \
			if [ $$? -eq 0 ]; then \
			[ `git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git status --porcelain -- src/$$of | wc -l` \
					-eq 0 ] || msg="not commited"; \
			else \
				msg="changed"; \
			fi; \
		else \
			msg="no file"; \
		fi; \
		if [ -n "$$msg" ]; then \
			diff="$$diff\t$$of: $$msg\n"; \
		fi; \
	done; \
	[ -n "$$diff" ] && printf "$*:\n$$diff"; \
	true
