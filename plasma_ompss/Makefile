# ------------------------------------------------------------------------------
# Usage:
#   make [all]      --  make lib test
#   make lib        --  make lib/libplasma.{a,so} lib/libcoreblas.{a,so}
#   make test       --  make test/test
#   make fortran_examples -- make executables in fortran_examples/
#   make docs       --  make docs/html
#   make generate   --  generate precisions
#   make clean      --  remove objects, libraries, and executables
#   make cleangen   --  remove generated precision files
#   make distclean  --  remove above, Makefile.*.gen, and anything else that can be generated
#
# Options:
#   quiet=1         --  abbreviate long compilation commands
#   fortran=0       --  do not build Fortran interface
#   fortran=1       --  build Fortran interface and examples


# ------------------------------------------------------------------------------
# Define default rule first, before including Makefiles

all: lib test


# ------------------------------------------------------------------------------
# Tools and flags
# Get from make.inc, or use these defaults

include make.inc

# define FPIC to make shared libraries
#FPIC      = -fPIC

CC        ?= cc
CFLAGS    ?= -std=c99 -fopenmp $(FPIC) -O3 -Wall -Wno-unused-variable -Wno-unused-function
LDFLAGS   ?= -fopenmp $(FPIC)

ARCH      ?= ar
ARCHFLAGS ?= cr
RANLIB    ?= ranlib

# INC and LIBS indicate where to find LAPACK, LAPACKE, and CBLAS
INC       ?= -I$(LAPACKDIR)/LAPACKE/include -I$(CBLASDIR)/include
LIBS      ?= -L$(LAPACKDIR) -llapack -llapacke -L$(CBLASDIR)/lib -lcblas -lblas

prefix    ?= /usr/local/plasma

# one of: aix bsd c89 freebsd generic linux macosx mingw posix solaris
# usually generic is fine


# ------------------------------------------------------------------------------
# Internal tools and flags

codegen     := ./tools/codegen.py

PLASMA_INC  := -Iinclude 
PLASMA_LIBS := -Llib -lplasma -lcoreblas

.DELETE_ON_ERROR:

.SUFFIXES:

ifeq ($(quiet),1)
   quiet_CC := @echo "$(CC) ... $@";
   quiet_FC := @echo "$(FC) ... $@";
   quiet_AR := @echo "$(ARCH) ... $@";
endif


# ------------------------------------------------------------------------------
# Define sources, objects, libraries, executables.
# These makefiles define lists of source and header files in
# $(plasma_all), $(coreblas_all), and $(test_all).

makefiles_gen := \
	Makefile.plasma.gen   \
	Makefile.coreblas.gen \
	Makefile.test.gen     
ifeq ($(fortran), 1)
	makefiles_gen += Makefile.fortran_examples.gen 
endif

-include $(makefiles_gen)

plasma_hdr   := $(filter %.h, $(plasma_all))
coreblas_hdr := $(filter %.h, $(coreblas_all))
test_hdr     := $(filter %.h, $(test_all))
headers      := $(plasma_hdr) $(coreblas_hdr) $(test_hdr)

plasma_obj   := $(addsuffix .o, $(basename $(filter-out %.h, $(plasma_all))))
coreblas_obj := $(addsuffix .o, $(basename $(filter-out %.h, $(coreblas_all))))
test_obj     := $(addsuffix .o, $(basename $(filter-out %.h, $(test_all))))

test_exe     := test/test

ifeq ($(fortran), 1)
    fortran_interface_src := include/plasma_mod.f90
    fortran_interface_obj := include/plasma_mod.o
    fortran_interface_mod := include/plasma.mod

    plasma_obj += $(fortran_interface_obj)

    fortran_examples_exe := $(basename $(fortran_examples_all))

all: fortran_examples
endif

# ------------------------------------------------------------------------------
# Create dependencies to generate Fortran interface.
fortran_gen := ./tools/fortran_gen.py
include/plasma_mod.f90: $(plasma_hdr)
	$(fortran_gen) --prefix include/ $^

$(fortran_interface_obj): $(fortran_interface_src)
	$(quiet_FC) $(FC) $(FFLAGS) $(PLASMA_INC) $(INC) -c -o $@ $<
	mv -f plasma.mod include/

# ------------------------------------------------------------------------------
# Build static libraries

libfiles += \
    lib/libplasma.a    \
    lib/libcoreblas.a  \

# In case changing Makefile.gen changes $(obj), also depend on it,
# which recreates the library if a file is removed.
lib/libplasma.a: $(plasma_obj) Makefile.plasma.gen
	mkdir -p lib
	-rm -f $@
	$(quiet_AR) $(ARCH) $(ARCHFLAGS) $@ $(plasma_obj)
	$(RANLIB) $@

lib/libcoreblas.a: $(coreblas_obj) Makefile.coreblas.gen
	mkdir -p lib
	-rm -f $@
	$(quiet_AR) $(ARCH) $(ARCHFLAGS) $@ $(coreblas_obj)
	$(RANLIB) $@


.PHONY: lib

lib: $(libfiles)


# ------------------------------------------------------------------------------
# Build tester

.PHONY: test

test: $(test_exe)

$(test_exe): $(test_obj) $(libfiles) Makefile.test.gen
	$(quiet_CC) $(CC) $(LDFLAGS) -o $@ $(test_obj) $(PLASMA_LIBS) $(LIBS) $(rpath)


# ------------------------------------------------------------------------------
# Build Fortran examples

.PHONY: fortran_examples

fortran_examples: $(fortran_examples_exe)

# implicit rule for building Fortran examples
% : %.f90 $(libfiles) Makefile.fortran_examples.gen
	$(quiet_FC) $(FC) $(FFLAGS) $(LDFLAGS) $(PLASMA_INC) $< -o $@ $(PLASMA_LIBS) $(LIBS) $(rpath)

run_fortran_tests: fortran_examples
	$(foreach exe,$(fortran_examples_exe), ./$(exe);)

# ------------------------------------------------------------------------------
# Build objects
# Headers must exist before compiling, but use order-only prerequisite
# (after "|") so as not to force recompiling everything if a header changes.
# (Should use compiler's -MMD flag to create header dependencies.)

%.o: %.c | $(headers)
	$(quiet_CC) $(CC) $(CFLAGS) $(PLASMA_INC) $(INC) -c -o $@ $<

%.i: %.c | $(headers)
	$(quiet_CC) $(CC) $(CFLAGS) $(PLASMA_INC) $(INC) -E -o $@ $<


# ------------------------------------------------------------------------------
# Build documentation

.PHONY: docs

docs: generate
	doxygen docs/doxygen/doxyfile.conf


# ------------------------------------------------------------------------------
# Install

.PHONY: install_dirs install

install_dirs:
	mkdir -p $(prefix)
	mkdir -p $(prefix)/include
	mkdir -p $(prefix)/lib
	mkdir -p $(prefix)/lib/pkgconfig

install: lib install_dirs
	cp include/*.h $(prefix)/include
	cp include/*.mod $(prefix)/include
	cp $(libfiles) $(prefix)/lib
	# pkgconfig
	cat lib/pkgconfig/plasma.pc.in         | \
	sed -e s:@INSTALL_PREFIX@:"$(prefix)": | \
	sed -e s:@CFLAGS@:"$(INC)":            | \
	sed -e s:@LIBS@:"$(LIBS)":             | \
	sed -e s:@REQUIRES@::                    \
	    > $(prefix)/lib/pkgconfig/plasma.pc

uninstall:
	rm -f $(prefix)/include/core_blas*.h
	rm -f $(prefix)/include/core_lapack*.h
	rm -f $(prefix)/include/plasma*.h
	rm -f $(prefix)/include/plasma.mod
	rm -f $(prefix)/lib/libcoreblas*
	rm -f $(prefix)/lib/libplasma*
	rm -f $(prefix)/lib/pkgconfig/plasma.pc


# ------------------------------------------------------------------------------
# Maintenance rules
# makefiles_gen define generate and cleangen.

.PHONY: clean distclean

clean:
	-rm -f $(plasma_obj) $(coreblas_obj) $(test_obj) $(test_exe) $(libfiles)
	-rm -f mcc_*.c
ifeq ($(fortran), 1)
	-rm -f $(fortran_interface_src) $(fortran_interface_obj) $(fortran_interface_mod)
	-rm -f $(fortran_examples_exe)
endif

# cleangen removes generated files if the template still exists;
# grep for any stale generated files without a template.
distclean: clean cleangen
	grep -s -l @generated $(plasma_src) $(coreblas_src) $(test_src) $(fortran_examples_src) | xargs rm -f
	-rm -f compute/*.o control/*.o core_blas/*.o test/*.o
	-rm -f $(makefiles_gen)
	-rm -rf docs/html


# ------------------------------------------------------------------------------
# Create dependencies to do precision generation.

plasma_src   := $(wildcard compute/*.c include/plasma*.h)

coreblas_src := $(wildcard core_blas/*.c control/*.c include/core*.h)

test_src     := $(wildcard test/*.c test/*.h)

fortran_examples_src := $(wildcard fortran_examples/*.f90)

Makefile.plasma.gen: $(codegen)
	$(codegen) --make --prefix plasma   $(plasma_src)   > $@

Makefile.coreblas.gen: $(codegen)
	$(codegen) --make --prefix coreblas $(coreblas_src) > $@

Makefile.test.gen: $(codegen)
	$(codegen) --make --prefix test     $(test_src)     > $@

Makefile.fortran_examples.gen: $(codegen)
	$(codegen) --make --prefix fortran_examples $(fortran_examples_src)     > $@

# --------------------
# If the list of src files changes, then force remaking Makefile.gen
# To reduce unnecesary remaking, don't remake if either:
# 1) src == old:
#    src has same files now as when Makefile.gen was generated, or
# 2) src - generated == templates:
#    src has all the templates from Makefile.gen, and no new non-generated files.
ifneq ($(plasma_src),$(plasma_old))
ifneq ($(filter-out $(plasma_generated),$(plasma_src)),$(plasma_templates))
Makefile.plasma.gen: force_gen
endif
endif

ifneq ($(coreblas_src),$(coreblas_old))
ifneq ($(filter-out $(coreblas_generated),$(coreblas_src)),$(coreblas_templates))
Makefile.coreblas.gen: force_gen
endif
endif

ifneq ($(test_src),$(test_old))
ifneq ($(filter-out $(test_generated),$(test_src)),$(test_templates))
Makefile.test.gen: force_gen
endif
endif

ifneq ($(fortran_examples_src),$(fortran_examples_old))
ifneq ($(filter-out $(fortran_examples_generated),$(fortran_examples_src)),$(fortran_examples_templates))
Makefile.fortran_examples.gen: force_gen
endif
endif
# --------------------

force_gen: ;


# ------------------------------------------------------------------------------
# Debugging

echo:
	@echo "CC      $(CC)"
	@echo "CFLAGS  $(CFLAGS)"
	@echo "LDFLAGS $(LDFLAGS)"
	@echo
	@echo "test_exe           <$(test_exe)>"
	@echo "libfiles           <$(libfiles)>"
	@echo
	@echo "plasma_src         <$(plasma_src)>"
	@echo "plasma_old         <$(plasma_old)>"
	@echo "plasma_templates   <$(plasma_templates)>"
	@echo "plasma_filtered    <$(filter-out $(plasma_generated),$(plasma))>"
	@echo "plasma_hdr         <$(plasma_hdr)>"
	@echo "plasma_obj         <$(plasma_obj)>"
	@echo
	@echo "coreblas_src       <$(coreblas_src)>"
	@echo "coreblas_old       <$(coreblas_old)>"
	@echo "coreblas_templates <$(coreblas_templates)>"
	@echo "coreblas_filtered  <$(filter-out $(coreblas_generated),$(coreblas))>"
	@echo "coreblas_hdr       <$(coreblas_hdr)>"
	@echo
	@echo "test_src           <$(test_src)>"
	@echo "test_old           <$(test_old)>"
	@echo "test_templates     <$(test_templates)>"
	@echo "test_filtered      <$(filter-out $(test_generated),$(test))>"
	@echo "test_hdr           <$(test_hdr)>"
	@echo
	@echo "fortran_examples_src       <$(fortran_examples_src)>"
	@echo "fortran_examples_exe       <$(fortran_examples_exe)>"
	@echo "fortran_examples_old       <$(fortran_examples_old)>"
	@echo "fortran_examples_templates <$(fortran_examples_templates)>"
	@echo "fortran_examples_filtered  <$(filter-out $(fortran_examples_generated),$(fortran_examples))>"
	@echo
	@echo "headers            <$(headers)>"
