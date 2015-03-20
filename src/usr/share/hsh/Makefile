LOCALCFG ?= Makefile.config

-include $(LOCALCFG)

DIR ?= src
POOLDIR ?= ""
TPLDIR ?= tpl
DIRS ?= $(shell find $(DIR) -maxdepth 1 -mindepth 1 -type d -exec basename '{}' \;)

SRC := $(DIRS:%=$(DIR)/%)
PWD := $(shell pwd)

.PHONY: all %/discard %/pack %/set-version %/set-branch %/check-modified %/divert
.PHONY: status %/status
.PHONY: diff %/diff

.PRECIOUS: $(SRC) $(SRC:%=%/.git)
.PRECIOUS: $(SRC:%=%/src/DEBIAN) $(SRC:%=%/src/DEBIAN/control)
.PRECIOUS: $(SRC:%=%/src/DEBIAN/postrm) $(SRC:%=%/src/DEBIAN/preinst)
.PRECIOUS: $(POOLDIR) $(POOLDIR)/Packages $(POOLDIR)/Packages.bz2

all: $(DIRS:%=%/discard)
	if [ -n $(POOLDIR) ]; then \
		dpkg-scanpackages $(POOLDIR) > $(POOLDIR)/Packages; \
		bzip2 -kf $(POOLDIR)/Packages; \
	fi

status: $(DIRS:%=%/status)

diff: $(DIRS:%=%/diff)

%/discard: %/pack
	git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git checkout -- src/DEBIAN/control

%/pack: %/set-version $(POOLDIR)
	rm -f $(POOLDIR)/$*_*.deb; \
	if [ -n $(POOLDIR) ]; then \
		dpkg-deb -b $(DIR)/$*/src $(POOLDIR); \
	else \
		dpkg-deb -b $(DIR)/$*/src .; \
	fi

$(POOLDIR):
	[ -n $(POOLDIR) ] && mkdir -p $(POOLDIR); \
	true

%/set-version: %/check-modified
	sed -i "s/^Version: {{git}}$$/Version: `git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git describe --tags \
	| sed 's/^[^0-9]*//'`/" $(DIR)/$*/src/DEBIAN/control;

%/check-modified: %/set-branch
	if [ ! $$(git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git status --porcelain | wc -l) -eq 0 ]; then \
		echo "$*: modified"; >&2 \
		exit 2; \
	fi

%/set-branch: $(DIR)/%/.git %/divert
	if [ $$(git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git branch | wc -l) -eq 0 ]; then \
		echo "$*: not initialized" >&2; \
		exit 1; \
	fi; \
	git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git checkout debian 2>&1; \
	[ $$? -eq 1 ] && git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git checkout -b debian 2>&1 && true; \
	true

$(DIR)/%/.git: $(DIR)/%/src/DEBIAN/control %/divert
	if [ ! -d $@ ]; then \
		cd $(DIR)/$* \
		&& git init \
		&& git add * \
		&& git ci -m 'initial commit' \
		&& git tag 0.0.0 \
		&& cd $(PWD); \
	fi

%/divert:
	@git="git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git"; \
	files=`find $(DIR)/$*/src -type f | grep -v DEBIAN | sed 's/$(DIR)\/$*\/src//'`; \
	div=""; \
	for f in $$files; do \
		if [ -e "$$f" ]; then \
			dpkg -S "$$f" 2>/dev/null 1>/dev/null; \
			if [ $$? -eq 0 ]; then\
				div="$$div $$f"; \
			fi; \
		fi; \
	done; \
	if [ -n "$$div" ]; then \
		for df in preinst postrm; do \
			of="$(DIR)/$*/src/DEBIAN/$$df"; \
			echo "#!/bin/sh\nset -e\n" > $$of; \
			if [ "$$df" = "preinst" ]; then \
				for f in $$div; do \
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

$(DIR)/%/src/DEBIAN/control: $(DIR)/%/src/DEBIAN
	if [ ! -e $@ ]; then \
		sed "s/Package: {{name}}/Package: $*/" $(TPLDIR)/DEBIAN/control \
		| sed "s/Description: {{description}}/Description: $*/" \
		| sed "s/Maintainer: {{maintainer}}/Maintainer: `git config --get user.name` <`git config --get user.email`>/" > $@; \
	fi

$(DIR)/%/src/DEBIAN:
	mkdir -p $@

%/status:
	@msg=`git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git status --porcelain`; \
	[ -n "$$msg" ] && printf "$*:\n$$msg\n"; \
	true

%/diff:
	@files=`find $(DIR)/$*/src -type f | grep -v DEBIAN | sed "s/$(DIR)\/$*\/src//"`; \
	diff=""; \
	for of in $$files; do \
		msg=""; \
		if [ -e "$$of" ]; then \
			diff $$of $(DIR)/$*/src/$$of > /dev/null; \
			if [ $$? -eq 0 ]; then \
			[ `git --work-tree $(DIR)/$* --git-dir $(DIR)/$*/.git status --porcelain -- src/$$of | wc -l` \
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
