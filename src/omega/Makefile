# BabelStream Omega model.
#
# The kernels are written in Omega (fmt.om + babelstream.om) and compiled by the seeded Omega
# compiler omc from https://github.com/aien-dev/omega-mv, pinned below by commit and sha256.
# omc runs on AArch64 Linux only and emits a static AArch64 Linux executable.
#
#   make                    fetch the pinned omc, check its sha256, build ./omega-stream
#   make OMC=/path/to/omc   build with an omc you already have (its sha256 is still checked)
#   make check              short runs that must pass check_solution
#   make negative-control   build a copy with a broken Triad kernel; it must fail check_solution

OMEGA_MV_COMMIT := e5b32567a7aff3ce594e3254a622a846a6cd190b
OMC_SHA256      := 2c82480596bbdc50cda9626866209ef662d5834f278f839ff4ce0250c495029f
OMC_URL         := https://raw.githubusercontent.com/aien-dev/omega-mv/$(OMEGA_MV_COMMIT)/seed/out/omc
OMC             ?= ./omc

SOURCES := fmt.om babelstream.om

.PHONY: all check negative-control clean

all: omega-stream

./omc:
	curl -fsSL --retry 3 -o omc.download $(OMC_URL)
	echo "$(OMC_SHA256)  omc.download" | sha256sum -c -
	chmod +x omc.download
	mv omc.download omc

# A compile is accepted only with exit status 0 and an empty stderr; a refused compile can still
# leave bytes on stdout, which are discarded.
define omc_compile
	echo "$(OMC_SHA256)  $(OMC)" | sha256sum -c --quiet -
	$(OMC) < $(1) > $(2).tmp 2> $(2).err || { cat $(2).err; rm -f $(2).tmp; exit 1; }
	if [ -s $(2).err ]; then cat $(2).err; rm -f $(2).tmp; exit 1; fi
	rm -f $(2).err
	chmod +x $(2).tmp
	mv $(2).tmp $(2)
endef

omega-stream.om: $(SOURCES)
	cat $(SOURCES) > $@

omega-stream: omega-stream.om $(OMC)
	$(call omc_compile,omega-stream.om,$@)

# Negative control: Triad writes b - scalar * c instead of b + scalar * c on the first pool.
omega-stream-mutant.om: $(SOURCES)
	sed 's/^put ad0 i add get bd0 i x$$/put ad0 i sub get bd0 i x/' babelstream.om | cat fmt.om - > $@
	test "$$(grep -c '^put ad0 i sub get bd0 i x$$' $@)" -eq 2

omega-stream-mutant: omega-stream-mutant.om $(OMC)
	$(call omc_compile,omega-stream-mutant.om,$@)

check: omega-stream
	./omega-stream --arraysize 2048
	./omega-stream --arraysize 2048 --float
	./omega-stream --arraysize 1048576 --numtimes 10 --only All
	./omega-stream --arraysize 1048576 --numtimes 10 --only All --float
	./omega-stream --arraysize 65537 --numtimes 5 --order Isolated --csv
	./omega-stream --arraysize 16777217 --numtimes 2 --only All
	./omega-stream --arraysize 16777216 --numtimes 2 --float

negative-control: omega-stream-mutant
	./omega-stream-mutant --arraysize 2048 --numtimes 4 > mutant.out 2> mutant.err; \
	rc=$$?; cat mutant.err; \
	test $$rc -eq 1 && grep -q '^FAILED validation of a\[0\]' mutant.err
	./omega-stream-mutant --arraysize 2048 --numtimes 4 --silence-errors > /dev/null 2>&1
	rm -f mutant.out mutant.err

clean:
	rm -f omega-stream omega-stream-mutant *.tmp *.err omega-stream.om omega-stream-mutant.om mutant.out omc.download
