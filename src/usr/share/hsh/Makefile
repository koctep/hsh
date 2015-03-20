HSHLOCALCFG ?= Makefile.config

-include $(HSHLOCALCFG)

HSHDIR ?= src
HSHPOOLDIR ?= ""
HSHPTLDIR ?= tpl
HSHDIRS ?= $(shell find $(HSHDIR) -maxdepth 1 -mindepth 1 -type d -exec basename '{}' \;)
HSHMODIFIED ?= 0
HSHCOLOR ?= 1

ifeq ($(HSHCOLOR), 1)
HSHCOLORPKT ?= \033[0;32m
HSHCOLORMSG ?= \033[0;33m
HSHCOLORRST = \033[00m
endif

SRC := $(HSHDIRS:%=$(HSHDIR)/%)
PWD := $(shell pwd)

.PHONY: all %/discard %/pack %/set-version %/set-branch %/check-modified %/divert %/divert-files
.PHONY: status %/status
.PHONY: diff %/diff

.PRECIOUS: $(SRC) $(SRC:%=%/.git)
.PRECIOUS: $(SRC:%=%/src/DEBIAN) $(SRC:%=%/src/DEBIAN/control)
.PRECIOUS: $(SRC:%=%/src/DEBIAN/postrm) $(SRC:%=%/src/DEBIAN/preinst)
.PRECIOUS: $(HSHPOOLDIR) $(HSHPOOLDIR)/Packages $(HSHPOOLDIR)/Packages.bz2

all: $(HSHDIRS:%=%/discard)
	@if [ -n $(HSHPOOLDIR) ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)updating packages file$(HSHCOLORRST)'; \
		dpkg-scanpackages $(HSHPOOLDIR) > $(HSHPOOLDIR)/Packages; \
		bzip2 -kf $(HSHPOOLDIR)/Packages; \
	fi

status: $(HSHDIRS:%=%/status)

diff: $(HSHDIRS:%=%/diff)

%/discard: %/pack
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)discarding changes$(HSHCOLORRST)'; \
	git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout -- src/DEBIAN/control; \
	dd=$(HSHDIR)/$*/src/DEBIAN; \
	[ -e $$dd/preinst ] && rm $$dd/preinst; \
	[ -e $$dd/postrm ] && rm $$dd/postrm; \
	[ -e $(HSHDIR)/$*/divert.files ] && rm $(HSHDIR)/$*/divert.files; \
	true

%/pack: %/set-version $(HSHPOOLDIR)
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)packing$(HSHCOLORRST)'; \
	if [ -n $(HSHPOOLDIR) ]; then \
		rm -f $(HSHPOOLDIR)/$*_*.deb; \
		fakeroot sh -c "chown -R root:root $(HSHDIR)/$*/src;dpkg-deb -b $(HSHDIR)/$*/src $(HSHPOOLDIR)"; \
	else \
		rm -f $*_*.deb; \
		fakeroot sh -c "chown -R root:root $(HSHDIR)/$*/src;dpkg-deb -b $(HSHDIR)/$*/src ."; \
	fi

$(HSHPOOLDIR):
	@[ -n $(HSHPOOLDIR) ] \
	&& echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating pooldir $(HSHPOOLDIR)$(HSHCOLORRST)' \
	&& mkdir -p $(HSHPOOLDIR); \
	true

%/set-version: %/divert
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting version$(HSHCOLORRST)'; \
	sed -i "s/^Version: {{git}}$$/Version: `git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git describe --tags \
	| sed 's/^[^0-9]*//'`/" $(HSHDIR)/$*/src/DEBIAN/control;

%/divert: %/divert-files
	@git="git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git"; \
	if [ -e "$(HSHDIR)/$*/divert.files" ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN/preinst|postrm$(HSHCOLORRST)'; \
		for df in preinst postrm; do \
			of="$(HSHDIR)/$*/src/DEBIAN/$$df"; \
			echo "#!/bin/sh\nset -e\n" > $$of; \
			if [ "$$df" = "preinst" ]; then \
				for f in `cat $(HSHDIR)/$*/divert.files`; do \
					echo "mkdir -p /var/backups`dirname $$f`\n" >> $$of; \
					echo "if [ \"\$$1\" = \"install\" ]; then" >> $$of;  \
					echo "\tdpkg-divert --package $* --rename \\" >> $$of; \
					echo "\t\t--divert /var/backups$$f $$f" >> $$of; \
					echo "fi" >> $$of; \
				done; \
			elif [ "$$df" = "postrm" ]; then \
				for f in `cat $(HSHDIR)/$*/divert.files`; do \
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
			[ -e $$of ] && rm $(HSHDIR)/$*/$$of; \
		done; \
	fi; \
	true

%/divert-files: %/check-modified
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diverting files$(HSHCOLORRST)'; \
	git="git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git"; \
	df=$(HSHDIR)/$*/divert ;\
	[ -e $$df.files ] && rm $$df.files;\
	[ -e $$df ] && cat $$df > $$df.files ;\
	files=`find $(HSHDIR)/$*/src -type f | grep -v DEBIAN | sed 's/$(HSHDIR)\/$*\/src//'`; \
	for f in $$files; do \
		if [ -e "$$f" ]; then \
			if [ `dpkg -S $$f 2>/dev/null | egrep -v '^$*' -c` -gt 0 ]; then\
				echo "$$f" >> $$df.files; \
			fi; \
		fi; \
	done; \

%/check-modified: %/set-branch
	@if [ $(HSHMODIFIED) -eq 0 ]; then \
		echo '$(HSHCOLORPKT)$*: $(HSHCOLORMSG)cheking repository modification$(HSHCOLORRST)'; \
		if [ ! $$(git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git status --porcelain | wc -l) -eq 0 ]; then \
			echo "$*: modified"; >&2 \
			exit 2; \
		fi; \
	fi

%/set-branch: $(HSHDIR)/%/.git
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting branch to "debian"$(HSHCOLORRST)'; \
	if [ $$(git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git branch | wc -l) -eq 0 ]; then \
		echo "$*: not initialized" >&2; \
		exit 1; \
	fi; \
	git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout debian 2>&1; \
	[ $$? -eq 1 ] && git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout -b debian 2>&1 && true; \
	true

$(HSHDIR)/%/.git: $(HSHDIR)/%/src/DEBIAN/control
	@if [ ! -d $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)git init && first commit$(HSHCOLORRST)'; \
		cd $(HSHDIR)/$* \
		&& git init \
		&& git add * \
		&& git ci -m 'initial commit' \
		&& git tag 0.0.0 \
		&& cd $(PWD); \
	fi

$(HSHDIR)/%/src/DEBIAN/control: $(HSHDIR)/%/src/DEBIAN
	@if [ ! -e $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting DEBIAN/control$(HSHCOLORRST)'; \
		sed "s/Package: {{name}}/Package: $*/" $(HSHPTLDIR)/DEBIAN/control \
		| sed "s/Description: {{description}}/Description: $*/" \
		| sed "s/Maintainer: {{maintainer}}/Maintainer: `git config --get user.name` <`git config --get user.email`>/" > $@; \
	fi

$(HSHDIR)/%/src/DEBIAN:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN directory$(HSHCOLORRST)'; \
	mkdir -p $@

%/status:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)status$(HSHCOLORRST)'; \
	msg=`git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git status --porcelain`; \
	[ -n "$$msg" ] && printf "$*:\n$$msg\n"; \
	true

%/diff:
	@echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diff$(HSHCOLORRST)'; \
	files=`find $(HSHDIR)/$*/src -type f | grep -v DEBIAN | sed "s/$(HSHDIR)\/$*\/src//"`; \
	diff=""; \
	for of in $$files; do \
		msg=""; \
		if [ -e "$$of" ]; then \
			diff $$of $(HSHDIR)/$*/src/$$of > /dev/null; \
			if [ $$? -eq 0 ]; then \
			[ `git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git status --porcelain -- src/$$of | wc -l` \
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
