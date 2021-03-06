MAKEFLAGS := rR
SHELL != which bash
is_parallel := $(if $(shell ps -p $${PPID} -o cmd|grep -P '\s+(--jobs|-\w*j(\s*[0-9]|\s|$$))'),1,)
project_root := $(realpath $(dir $(lastword $(MAKEFILE_LIST)))/..)
maybe_devnull := $(if $(VERBOSE),,>/dev/null)
maybe_quiet := $(if $(VERBOSE),,-q)
st_root := $(project_root)/test/st
st_common := $(st_root)/common
systems := $(filter-out common,$(notdir $(wildcard $(st_root)/*)))
st := $(sort $(subst /,+,$(patsubst $(st_root)/%.sh,%,$(foreach f,$(systems),$(wildcard $(st_root)/$f/*.sh)))) \
             $(foreach s,$(systems),$s+$(patsubst %.sh,%,$(notdir $(wildcard $(st_common)/*.sh)))))

.PHONY: all clean sa st $(st) ut
.$(VERBOSE)SILENT:
.DELETE_ON_ERROR:
.INTERMEDIATE: $(project_root)/openvpn-unroot.bash

all: sa ut st

sa: $(project_root)/openvpn-unroot
	shellcheck -s bash "$<"

$(project_root)/openvpn-unroot.bash: %.bash: %
	ln -s "$<" "$@"

ut: | $(project_root)/openvpn-unroot.bash
	! $(project_root)/test/ut/ut.bats $(UT_OPTS)|grep -P '^(not|#)'

st: $(st)

$(st_common)/vpngate.csv:
	curl -s http://www.vpngate.net/api/iphone/|tail -n +3|head -n -1 >"$@"

# Random config from vpngate.net.
$(st_common)/vpngate.conf: | $(st_common)/vpngate.csv
	shuf -n1 "$|"|cut -d, -f15|tr -d '\r'|base64 --decode >"$@"

# Docker stuff all the way down.
$(filter %+vpngate,$(st)): $(st_common)/vpngate.conf
docker_targets := $(addsuffix +%,$(systems))
$(docker_targets) $(st): system = $(word 1,$(subst +, ,$@))
$(docker_targets) $(st): script = $(word 2,$(subst +, ,$@))

.SECONDEXPANSION:
docker_dep = $(st_root)/$$(system)/Dockerfile
define docker_recipe
	docker build $(maybe_quiet) -t "openvpn-unroot/$(system):latest" -f "$<" "$(dir $<)" $(maybe_devnull)
	docker run --rm --cap-add=NET_ADMIN --device /dev/net/tun -v "$(project_root)/openvpn-unroot:/usr/bin/openvpn-unroot" \
	           $$(for f in $(st_root)/common/* $(st_root)/$(system)/*; do echo -v "$$f:/tmp/test/$$(basename $$f)"; done) \
	           $(if $(is_parallel),,-it )-w "/tmp/test" openvpn-unroot/$(system):latest \
	           $(if $(wildcard $(st_root)/*/$(script).sh $(st_common)/$(script).sh),\
	               "./$(script).sh" $(if $(VERBOSE),verbose,) $(maybe_devnull),\
	               $(script) $(foreach a,$(EXE_ARGS),"$(a)") $(if $(EXE_ARG),"$(EXE_ARG)",))
endef

# Any target based on docker.
$(docker_targets): $(docker_dep); $(docker_recipe)

# Docker-based STs.
$(st): $(docker_dep); $(docker_recipe)

clean:
	rm -f "$(project_root)/openvpn-unroot.bash" "$(st_common)/vpngate.csv" "$(st_common)/vpngate.conf"
