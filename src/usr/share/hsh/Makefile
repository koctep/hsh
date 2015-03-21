HSHLOCALCFG ?= Makefile.config
-include $(HSHLOCALCFG)

PWD := $(shell pwd)

ifeq ($(hsh), packet)
HSHROOT 		:= ..
HSHPACKETS 	:= $(shell basename $(PWD))
endif

ifeq ($(hsh), dir)
HSHROOT			:= .
endif

HSHTPLDIR ?= tpl

HSHMODIFIED ?= 0

HSHCOLOR ?= 1

HSHUPDATEDIVERT ?= 0

HSHROOT ?= src
HSHPACKETS ?= $(shell find $(HSHROOT) -maxdepth 1 -mindepth 1 -type d -exec basename '{}' \;)

HSHPOOLDIR ?= ""

ifeq ($(HSHPOOLDIR), "")
HSHDEST ?= $(HSHROOT)
else
HSHDEST ?= $(HSHPOOLDIR)
endif

ifeq ($(HSHCOLOR), 1)
HSHCOLORPKT ?= \033[0;32m
HSHCOLORMSG ?= \033[0;33m
HSHCOLORALERT ?= \033[0;31m
HSHCOLORRST = \033[00m
endif

ifneq ($(HSHPRINTCMDS), 1)
HSHCMDPREFIX := @
endif

HSHFORCEUPDATE ?= 0
HSHUPDATEONDIFF ?= 0
HSHFAILONDIFF ?= 1

ifeq ($(HSHFORCEUPDATE), 1)
HSHUPDATEONDIFF := 1
endif

ifeq ($(HSHUPDATEONDIFF), 1)
HSHFAILONDIFF := 0
endif

HSHCOMMIT := $(HSHCOLORALERT)hsh commit$(HSHCOLORRST)

SRC := $(HSHPACKETS:%=$(HSHROOT)/%)

.PHONY: all opts
.PHONY: all %/discard %/pack
.PHONY: %/set-version %/set-depends %/set-maintainer %/divert %/divert-files
.PHONY: %/set-branch %/check-modified
.PHONY: status %/status
.PHONY: diff %/diff

.PRECIOUS: $(SRC) $(SRC:%=%/.git)
.PRECIOUS: $(SRC:%=%/src/DEBIAN) $(SRC:%=%/src/DEBIAN/control)
.PRECIOUS: $(SRC:%=%/src/DEBIAN/postrm) $(SRC:%=%/src/DEBIAN/preinst)
.PRECIOUS: $(HSHPOOLDIR) $(HSHPOOLDIR)/Packages $(HSHPOOLDIR)/Packages.bz2

all: $(HSHPACKETS:%=%/discard)
	$(HSHCMDPREFIX)if [ -n $(HSHPOOLDIR) ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)updating packages file$(HSHCOLORRST)'; \
		dpkg-scanpackages $(HSHPOOLDIR) > $(HSHPOOLDIR)/Packages; \
		bzip2 -kf $(HSHPOOLDIR)/Packages; \
	fi

status: $(HSHPACKETS:%=%/status)

diff: $(HSHPACKETS:%=%/diff)

opts:
	$(HSHCMDPREFIX)echo "pooldir: $(HSHPOOLDIR)"; \
	echo "dir: $(HSHROOT)"; \
	echo "packets: $(HSHPACKETS)"

%/discard: %/pack
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)discarding changes$(HSHCOLORRST)'; \
	git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout -- src/DEBIAN/control; \
	debdir=$(HSHROOT)/$*/src/DEBIAN; \
	[ -e $$debdir/preinst ] && rm $$debdir/preinst; \
	[ -e $$debdir/postrm ] && rm $$debdir/postrm; \
	true

%/pack: $(HSHPOOLDIR) %/set-version
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)packing$(HSHCOLORRST)'; \
	mkdir -p $(HSHDEST); \
	rm -f $(HSHDEST)/$*_*.deb; \
	fakeroot sh -c "chown -R root:root $(HSHROOT)/$*/src;dpkg-deb -b $(HSHROOT)/$*/src $(HSHDEST)";

$(HSHPOOLDIR):
	$(HSHCMDPREFIX)[ -n $(HSHPOOLDIR) ] \
	&& echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating dir $(HSHPOOLDIR)$(HSHCOLORRST)' \
	&& mkdir -p $(HSHPOOLDIR); \
	true

%/set-version: %/set-depends
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting version$(HSHCOLORRST)'; \
	git="git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git"; \
	ver=`$$git describe --tags | sed 's/^[^0-9]*//'`; \
	sed -i "s/^Version: {{git}}$$/Version: $$ver/" $(HSHROOT)/$*/src/DEBIAN/control;

%/set-depends: %/set-maintainer
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting dependencies$(HSHCOLORRST)'; \
	deps=""; \
	if [ -e $(HSHROOT)/$*/depends ]; then \
		p=""; \
		for d in `cat $(HSHROOT)/$*/depends`; do \
			deps="$$deps$$p$$d"; \
			p=", "; \
		done; \
	fi; \
	sed -i "s/Depends: {{depends}}/Pre-Depends: $$deps/" $(HSHROOT)/$*/src/DEBIAN/control

%/set-maintainer: %/divert
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting maintainer$(HSHCOLORRST)'; \
	sed -i "s/Maintainer: {{maintainer}}/Maintainer: `git config --get user.name` <`git config --get user.email`>/" $(HSHROOT)/$*/src/DEBIAN/control

%/divert: %/divert-files
	$(HSHCMDPREFIX)git="git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git"; \
	if [ -e "$(HSHROOT)/$*/divert.files" ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN/preinst|postrm$(HSHCOLORRST)'; \
		for df in preinst postrm; do \
			of="$(HSHROOT)/$*/src/DEBIAN/$$df"; \
			echo "#!/bin/sh\nset -e\n" > $$of; \
			if [ "$$df" = "preinst" ]; then \
				for f in `sort -u $(HSHROOT)/$*/divert.files`; do \
					echo "mkdir -p /var/backups`dirname $$f`\n" >> $$of; \
					echo "if [ \"\$$1\" = \"install\" ]; then" >> $$of;  \
					echo "\tdpkg-divert --package $* --rename \\" >> $$of; \
					echo "\t\t--divert /var/backups$$f $$f" >> $$of; \
					echo "fi" >> $$of; \
				done; \
			elif [ "$$df" = "postrm" ]; then \
				for f in `sort -u $(HSHROOT)/$*/divert.files`; do \
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
			[ -e $(HSHROOT)/$*/$$of ] && rm $(HSHROOT)/$*/$$of; \
		done; \
	fi; \
	true

%/divert-files: %/check-modified %/diff
	$(HSHCMDPREFIX)git="git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git"; \
	df=$(HSHROOT)/$*/divert; \
	if [ $(HSHUPDATEDIVERT) -eq 1 ] \
		|| ([ -e "$$df" ] && [ ! -e "$$df.files" ]) \
		|| ([ `dpkg --get-selections | egrep '$*.*install' | grep -v 'deinstall' -c` -eq 1 ]) \
		; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diverting files$(HSHCOLORRST)'; \
		[ -e "$$df.files" ] && rm "$$df.files"; \
		[ -e "$$df" ] && cat "$$df" > "$$df.files"; \
		files=`find $(HSHROOT)/$*/src -type f | grep -v DEBIAN | sed 's/$(HSHROOT)\/$*\/src//'`; \
		for f in $$files; do \
			if [ -e "$$f" ]; then \
				if [ `dpkg -S $$f 2>/dev/null | egrep -v '^$*' -c` -gt 0 ]; then\
					echo "$$f" >> $$df.files; \
				fi; \
			fi; \
		done; \
		if [ -e $$df.files ]; then \
			echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORALERT)adding divert.files$(HSHCOLORRST)' >&2; \
			msg=`cat $$df.files`; \
			msg=`echo "\n$$msg"`; \
			$$git add divert.files; \
			$$git commit -m "$(HSHCOMMIT):\ndiverting files:$$msg"; \
		else \
			echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORALERT)removing divert.files$(HSHCOLORRST)' >&2; \
			if [ `$$git ls-files divert.files | wc -l` -eq 1 ]; then \
				$$git rm divert.files; \
				$$git commit -m "$(HSHCOMMIT): removed divert.files"; \
			fi; \
		fi; \
		true; \
	fi;

%/check-modified: %/set-branch
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)cheking repository modification$(HSHCOLORRST)'; \
	if [ ! $$(git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git status --porcelain | wc -l) \
			-eq 0 ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORALERT)modified$(HSHCOLORRST)' >&2; \
		if [ $(HSHMODIFIED) -eq 0 ]; then \
			exit 2; \
		fi; \
	fi

%/set-branch: $(HSHROOT)/%/.git
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting branch to "debian"$(HSHCOLORRST)'; \
	if [ $$(git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git branch | wc -l) -eq 0 ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORALERT)not initialized$(HSHCOLORRST)' >&2; \
		exit 1; \
	fi; \
	git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout debian 2>&1; \
	[ $$? -eq 1 ] && git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git checkout -b debian 2>&1 && true; \
	true

$(HSHROOT)/%/.git: $(HSHROOT)/%/src/DEBIAN/control
	$(HSHCMDPREFIX)if [ ! -d $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)git init && first commit$(HSHCOLORRST)'; \
		cd $(HSHROOT)/$* \
		&& git init \
		&& git add * \
		&& git commit -m '$(HSHCOMMIT): initial commit' \
		&& git tag 0.0.0 \
		&& cd $(PWD); \
	fi

$(HSHROOT)/%/src/DEBIAN/control: $(HSHROOT)/%/src/DEBIAN
	$(HSHCMDPREFIX)if [ ! -e $@ ]; then \
		echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)setting DEBIAN/control$(HSHCOLORRST)'; \
		sed "s/Package: {{name}}/Package: $*/" $(HSHTPLDIR)/DEBIAN/control \
		| sed "s/Description: {{description}}/Description: $*/" \
		> $@; \
	fi

$(HSHROOT)/%/src/DEBIAN:
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)creating DEBIAN directory$(HSHCOLORRST)'; \
	mkdir -p $@

%/status:
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)status$(HSHCOLORRST)'; \
	msg=`git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git status --porcelain`; \
	[ -n "$$msg" ] && printf "$*:\n$$msg\n"; \
	true

%/diff:
	$(HSHCMDPREFIX)echo "\t"'$(HSHCOLORPKT)$*: $(HSHCOLORMSG)diff$(HSHCOLORRST)'; \
	pn=`dpkg --get-selections | egrep '$*.*install' | grep -v 'deinstall' -c`; \
	diff=""; \
	gitc=""; \
	git='git --work-tree $(HSHROOT)/$* --git-dir $(HSHROOT)/$*/.git'; \
	if ([ $$pn -eq 1 ] || [ $(HSHFORCEUPDATE) -eq 1 ]) && [ ! "$*" = "hsh" ]; then \
		files=`find $(HSHROOT)/$*/src -type f | grep -v DEBIAN | sed "s/$(HSHROOT)\/$*\/src//"`; \
		for of in $$files; do \
			msg=""; \
			if [ -e "$$of" ]; then \
				diff $$of $(HSHROOT)/$*/src/$$of > /dev/null; \
				if [ ! $$? -eq 0 ]; then \
					if [ $(HSHUPDATEONDIFF) -eq 1 ]; then \
						cp $$of $(HSHROOT)/$*/src/$$of; \
					fi; \
					msg="different in system"; \
				fi; \
			else \
				msg="$(HSHCOLORMSG)no file in system, $(HSHCOLORALERT)outdated packet $(HSHCOLORPKT)$*$(HSHCOLORRST)"; \
			fi; \
				if [ `$$git status --porcelain -- src/$$of | wc -l` -gt 0 ]; then \
					if [ $(HSHUPDATEONDIFF) -eq 1 ]; then \
						$$git add src/$$of; \
						msg="$$msg, committing"; \
					else \
						msg="$$msg, not committed"; \
					fi; \
				fi; \
			if [ -n "$$msg" ]; then \
				diff="\n$$diff\t$$of: $(HSHCOLORMSG)$$msg$(HSHCOLORRST)"; \
				gitc="\n$$gitc\t$$of"; \
			fi; \
		done; \
	elif [ $$pn -gt 1 ]; then \
		diff="$(HSHCOLORALERT)matched several packets$(HSHCOLORRST)"; \
	fi; \
	if [ -n "$$diff" ]; then \
		echo "$*:$$diff"; \
	fi; \
	if [ -n "$$gitc" ]; then \
		$$git commit -m "`echo \"$(HSHCOMMIT):$$gitc\"`"; \
	fi; \
	if [ -n "$$diff" ] && [ $(HSHFAILONDIFF) -eq 1 ]; then \
		exit 1; \
	fi; \
	true
