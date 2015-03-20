HSHLOCALCFG ?= Makefile.config

-include $(HSHLOCALCFG)

HSHDIR ?= src
HSHPOOLDIR ?= ""
HSHPTLDIR ?= tpl
HSHDIRS ?= $(shell find $(HSHDIR) -maxdepth 1 -mindepth 1 -type d -exec basename '{}' \;)

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
	if [ -n $(HSHPOOLDIR) ]; then \
		dpkg-scanpackages $(HSHPOOLDIR) > $(HSHPOOLDIR)/Packages; \
		bzip2 -kf $(HSHPOOLDIR)/Packages; \
	fi

status: $(HSHDIRS:%=%/status)

diff: $(HSHDIRS:%=%/diff)

%/discard: %/pack
	git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout -- src/DEBIAN/control

%/pack: %/set-version $(HSHPOOLDIR) %/divert
	if [ -n $(HSHPOOLDIR) ]; then \
		rm -f $(HSHPOOLDIR)/$*_*.deb; \
		fakeroot sh -c "chown -R root:root $(HSHDIR)/$*/src;dpkg-deb -b $(HSHDIR)/$*/src $(HSHPOOLDIR)"; \
	else \
		rm -f $*_*.deb; \
		fakeroot sh -c "chown -R root:root $(HSHDIR)/$*/src;dpkg-deb -b $(HSHDIR)/$*/src ."; \
	fi

$(HSHPOOLDIR):
	[ -n $(HSHPOOLDIR) ] && mkdir -p $(HSHPOOLDIR); \
	true

%/set-version: %/check-modified
	sed -i "s/^Version: {{git}}$$/Version: `git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git describe --tags \
	| sed 's/^[^0-9]*//'`/" $(HSHDIR)/$*/src/DEBIAN/control;

%/divert: %/divert-files
	@git="git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git"; \
	if [ -e "$(HSHDIR)/$*/divert.files" ]; then \
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
				for f in $$div; do \
					echo "if [ remove = \"\$$1\"" \
						" -o abort-install = \"\$$1\"" \
						" -o disappear = \"\$$1\" ]; then" >> $$of; \
					echo "\tdpkg-divert --package $* --rename --remove $$f" >> $$of; \
					echo "fi" >> $$of; \
				done; \
			fi ;\
			echo "\nexit 0" >> $$of; \
			chmod a+x $$of; \
			$$git add src/DEBIAN/$$df; \
		done; \
		$$git commit -m 'added preinst/postrm hooks'; \
	else \
		for df in preinst postrm; do \
			of="src/DEBIAN/$$df"; \
			[ -e $$of ] && $$git rm $$of; \
		done; \
		$$git commit -m 'deleted preinst/postrm hooks'; \
	fi; \
	true

%/divert-files:
	@git="git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git"; \
	df=$(HSHDIR)/$*/divert ;\
	[ -e $$df.files ] && rm $$df.files;\
	[ -e $$df ] && cat $$df > $$df.files ;\
	files=`find $(HSHDIR)/$*/src -type f | grep -v DEBIAN | sed 's/$(HSHDIR)\/$*\/src//'`; \
	for f in $$files; do \
		if [ -e "$$f" ]; then \
			dpkg -S "$$f" 2>/dev/null 1>/dev/null; \
			if [ $$? -eq 0 ]; then\
				echo "$$f" >> $$df; \
			fi; \
		fi; \
	done; \

%/check-modified: %/set-branch
	if [ ! $$(git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git status --porcelain | wc -l) -eq 0 ]; then \
		echo "$*: modified"; >&2 \
		exit 2; \
	fi

%/set-branch: $(HSHDIR)/%/.git %/divert
	if [ $$(git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git branch | wc -l) -eq 0 ]; then \
		echo "$*: not initialized" >&2; \
		exit 1; \
	fi; \
	git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout debian 2>&1; \
	[ $$? -eq 1 ] && git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git checkout -b debian 2>&1 && true; \
	true

$(HSHDIR)/%/.git: $(HSHDIR)/%/src/DEBIAN/control
	if [ ! -d $@ ]; then \
		cd $(HSHDIR)/$* \
		&& git init \
		&& git add * \
		&& git ci -m 'initial commit' \
		&& git tag 0.0.0 \
		&& cd $(PWD); \
	fi

$(HSHDIR)/%/src/DEBIAN/control: $(HSHDIR)/%/src/DEBIAN
	if [ ! -e $@ ]; then \
		sed "s/Package: {{name}}/Package: $*/" $(HSHPTLDIR)/DEBIAN/control \
		| sed "s/Description: {{description}}/Description: $*/" \
		| sed "s/Maintainer: {{maintainer}}/Maintainer: `git config --get user.name` <`git config --get user.email`>/" > $@; \
	fi

$(HSHDIR)/%/src/DEBIAN:
	mkdir -p $@

%/status:
	@msg=`git --work-tree $(HSHDIR)/$* --git-dir $(HSHDIR)/$*/.git status --porcelain`; \
	[ -n "$$msg" ] && printf "$*:\n$$msg\n"; \
	true

%/diff:
	@files=`find $(HSHDIR)/$*/src -type f | grep -v DEBIAN | sed "s/$(HSHDIR)\/$*\/src//"`; \
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
