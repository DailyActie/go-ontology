all: stage_release

# ----------------------------------------
# CORE VARIABLES
# ----------------------------------------

# base URI for all OBO library ontologies
OBO=http://purl.obolibrary.org/obo

# OBO name of this ontology (always lowercase)
ONT=go

# Base URL for ontology
BASE=$(OBO)/$(ONT)

# Source file edited by editors
SRC=$(ONT)-edit.obo

# Staging release directory
RELEASEDIR=../../target
RELEASEDIR_IMPORTS=$(RELEASEDIR)/imports

# ROBOT is the main release release tool
ROBOT_ENV = ROBOT_JAVA_ARGS=-Xmx8G
ROBOT= $(ROBOT_ENV) robot

# OWLTools is a swiss-army knife;
# It does some things robot can't do, but eventually functionality should
# move to robot
OWLTOOLS_ENV = OWLTOOLS_MEMORY=8G
OWLTOOLS= $(OWLTOOLS_ENV) owltools

# Oort build dir;
# This also should be replaced by robot
BUILD_DIR=build

# OWL catalogs map ontology IRIs to local paths
USECAT= --use-catalog

# obo-centric checks and and filters
PERLCHECK= ../util/check-obo-for-standard-release.pl
PERLFILTER= ../util/filter-obo-for-standard-release.pl

# Release IRIs are of the form
#  obo/go/releases/YYYY-MM-DD/$FILE
#
# To make snapshot releases:
#
#   $ RELEASE_SUFFIX=SNAPSHOT make prepare_release
#
RELEASE_SUFFIX=
RELEASE_URIBASE= $(BASE)/releases/`date +%Y-%m-%d`$(RELEASE_SUFFIX)

# basic relations are part_of & 3 regulations relations
GO_BASIC_RELATIONS = BFO:0000050 RO:0002211 RO:0002212 RO:0002213

# extend with has_part, occurs_in and during relations
# - this is the set used in the main go file
GO_MAIN_RELATIONS = $(GO_BASIC_RELATIONS) BFO:0000051 BFO:0000066 RO:0002091 RO:0002092 RO:0002093

GO_PLUS = extensions/go-plus

SPARQLDIR = ../sparql
REPORTDIR = $(RELEASEDIR)/reports

# ----------------------------------------
# TOP LEVEL TARGETS
# ----------------------------------------

stage_release: test $(ONT).owl $(ONT).obo $(ONT).json.gz $(GO_PLUS).owl $(GO_PLUS).json.gz go-basic.obo go-basic.json.gz subset-reports

test: $(SRC)-check sparql_test change-report.txt reasoned.owl

# note: until we go live, the travis test must omit change-report. This is because as a demo repo,
# we naturally fall behind the actual live PURL. **DELETE THIS TARGET WHEN WE GO LIVE**
#travis_test: $(SRC)-check sparql_test reasoned.owl
travis_test: $(SRC)-check sparql_test change-report.txt all_reports reasoned.owl

full_test: test $(ONT).obo

# note that most derived files are kept separate from source files, in $(RELEASEDIR)
clean: clean_target clean_imports
clean_target:
	rm -rf $(RELEASEDIR)/*

# copies from staged area to target
prepare_release: build
	mkdir -p $(RELEASEDIR)/extensions &&\
	mkdir -p $(RELEASEDIR)/subsets &&\
	mkdir -p $(RELEASEDIR)/reports &&\
	mkdir -p $(RELEASEDIR_IMPORTS) &&\
	cp $(ONT).owl $(ONT).obo $(ONT).json $(RELEASEDIR) &&\
	cp imports/*.owl $(RELEASEDIR_IMPORTS) &&\
	cp imports/*.obo $(RELEASEDIR_IMPORTS) &&\
	cp extensions/* $(RELEASEDIR)/extensions &&\
	cp go-basic.* $(RELEASEDIR) &&\
	cp subsets/* $(RELEASEDIR)/subsets &&\
	cp reports/* $(RELEASEDIR)/reports &&\
	echo "Release files are now in $(RELEASEDIR)"

# deploy to S3
# URLs will be of the form https://s3.amazonaws.com/go-data-product-current/ontology/subsets/go-basic.json.gz
S3CFG = ~/.s3cfg.go-pipeline-push-agent
RELEASEDATE = `date +%Y-%m-%d`
# https://build.berkeleybop.org/job/release-go-ontology-daily-snapshot/
deploy-snapshot:
	cd $(RELEASEDIR) &&\
	s3cmd -c $(S3CFG) --acl-public --mime-type=application/rdf+xml sync ./  s3://go-data-product-snapshot/ontology/

deploy-release:
	cd $(RELEASEDIR) &&\
	s3cmd -c $(S3CFG) --acl-public --mime-type=application/rdf+xml sync ./  s3://go-data-product-current/ontology/
	s3cmd -c $(S3CFG) --acl-public --mime-type=application/rdf+xml sync ./  s3://go-data-product-release/$(RELEASEDATE)/ontology/


# ----------------------------------------
# GENERIC TRANSFORMS
# ----------------------------------------

%.gz: %
	gzip -c $< > $@.tmp && mv $@.tmp $@

%.json: %.owl
	$(OWLTOOLS) $(USECAT) $< -o -f json $@
.PRECIOUS: %.json

# ----------------------------------------
# CHECKS AND BALANCES
# ----------------------------------------

# TODO: use json form of this file
# See also: https://github.com/geneontology/go-ontology/issues/13837
GO.xrf_abbs: $(SRC)
	wget --no-check-certificate https://s3.amazonaws.com/go-public/metadata/db-xrefs.legacy -O $@.tmp && mv $@.tmp $@ && touch $@

$(SRC)-check: $(SRC) GO.xrf_abbs
	$(PERLCHECK) --disable-isa-incomplete --xref-abbs GO.xrf_abbs $< > $@.tmp && mv $@.tmp $@

# consider using snapshot release
PREVIOUS=go-lastrelease.owl
$(PREVIOUS): $(SRC)
	wget $(OBO)/go.obo -O $@ && touch $@

# ensure no IDs are lost
change-report.txt: $(SRC) $(PREVIOUS)
	$(OWLTOOLS) $(USECAT) $< --verify-changes -p $(PREVIOUS) -o $@ --id-prefix-filter GO: --check-missing-labels && touch $@

# ----------------------------------------
# OWL
# ----------------------------------------

# Translate never_in_taxon annotation assertion axioms to
# logical disjointness axioms
disjoint_in_taxon.owl: $(SRC)
	robot query -i $< --construct $(SPARQLDIR)/constructDisjointInTaxon.sparql $@ -f ttl

# merge in any CONSTRUCTs
enhanced.owl: $(SRC) disjoint_in_taxon.owl
	$(OWLTOOLS) $(USECAT) $^ --merge-support-ontologies -o $@

# reasoned.owl is equivalent to editors source, after reason-relax-reduce pipeline
# note: we keep this as a distinct intermediate target as the ontology IRI needs to be 'go' for Oort to work...
reasoned.owl: enhanced.owl $(SRC)-check
	$(ROBOT)  reason -i $< -r ELK relax reduce  annotate -V $(RELEASE_URIBASE)/$(ONT).owl -o $@

# equivalent to reasoned, but we rename
$(GO_PLUS).owl: reasoned.owl
	$(ROBOT) annotate -i $< -O $(BASE)/$@ -V $(RELEASE_URIBASE)/$@ -o $@
.PRECIOUS: $(GO_PLUS).owl

# ----------------------------------------
# OBO and subsets
# ----------------------------------------

# while go-plus is the full version, the successive releases eliminate various things to simplify the
# file, including
#  - no imports or external ontologies
#  - smaller set of relations (v small for go-basic)
#  - ...
# go.obo is a simplified; we use oort to remove imports etc

$(ONT)-pre.obo: reasoned.owl
	owltools $(USECAT) $< --remove-imports-declarations --remove-dangling --make-subset-by-properties $(GO_MAIN_RELATIONS) --set-ontology-id $(OBO)/go.owl -o -f obo $@.tmp && egrep -v '(property_value|relationship): never_in_taxon' $@.tmp > $@

$(ONT)-pre-reduced.obo: $(ONT)-pre.obo
	robot reduce -r elk -i $< -o $@
.PRECIOUS: $(ONT)-pre-reduced.obo

# use OORT
# side effects: also builds go-simple.obo
$(ONT).obo: $(ONT)-pre-reduced.obo
	ontology-release-runner --catalog-xml catalog-v001.xml --ignoreLock --skip-release-folder --skip-format owx --skip-format metadata --outdir $(BUILD_DIR) --allow-overwrite --asserted --simple --no-reasoner --remove-trailing-qualifiers $< &&\
	$(PERLCHECK) --disable-xrf-abbs-check --disable-multiply-labeled-edge $(BUILD_DIR)/go.obo &&\
	cp $(BUILD_DIR)/subsets/* subsets/ &&\
	cp $(BUILD_DIR)/go.obo $@ &&\
	cp $(BUILD_DIR)/go-simple.obo . &&\
	cp $(BUILD_DIR)/go.owl .


# This is produced as a byproduct of go.obo
go-simple.obo: $(ONT).obo
go.owl: $(ONT).obo

# remove non-basic relationship types (e.g. has_part)
go-simple-filtered.obo: go-simple.obo
	owltools $< --make-subset-by-properties $(GO_BASIC_RELATIONS) // -o -f obo $@

go-simple-filtered-reduced.obo: go-simple-filtered.obo
	robot reduce -r elk -i $< -o $@

# filter inter-ontology (e.g. MF to BP) links
go-basic.obo: go-simple-filtered-reduced.obo
	$(PERLFILTER) $< > $@.tmp && mv $@.tmp $@ && $(PERLCHECK) --disable-xrf-abbs-check $@

go-basic.owl: go-basic.obo
	$(ROBOT) convert -i $< -o $@

# ----------------------------------------
# SUBSET REPORTS
# ----------------------------------------
# Currently the only per-subset report is id-label pairs
# See: https://github.com/geneontology/go-ontology/issues/14028

# get all subsets
subset-files = $(shell ls subsets/*.owl)

# so far we only make simple .tsv reports
SUBSET_REPORTS = $(patsubst %,%.tsv,$(basename $(subset-files)))
subset-reports: $(SUBSET_REPORTS)
subsets/%.tsv: subsets/%.owl
	robot query -f tsv -i $< -s ../sparql/labels.sparql $@

# ----------------------------------------
# IMPORT MODULES
# ----------------------------------------

IMPORTS = chebi cl po pato ro ncbitaxon uberon ddanat fao oba so
IMPORTS_OWL = $(patsubst %, imports/%_import.owl,$(IMPORTS)) $(patsubst %, imports/%_import.obo,$(IMPORTS))

# Make this target to regenerate ALL
all_imports: $(IMPORTS_OWL)
clean_imports:
	rm mirror/*

seed.txt: enhanced.owl
	$(ROBOT) query -i $< --select ../sparql/terms.sparql $@

imports/%_terms_combined.txt: imports/%_terms.txt seed.txt
	cat $^ > $@

# Use ROBOT for imports
imports/%_import_pre.owl: mirror/%.owl imports/%_terms_combined.txt $(SRC)
	$(ROBOT) extract -i $< -T imports/$*_terms_combined.txt --method BOT -O $(BASE)/imports/$*_import.owl annotate -V $(RELEASE_URIBASE)/imports/$*_import.owl -o $@
.PRECIOUS: imports/%_import_pre.owl

imports/ncbitaxon_import.owl: imports/ncbitaxon_import_pre.owl
	owltools $< --create-taxon-disjoint-over-in-taxon -s -r NCBITaxon:1 -m --set-ontology-id $(RELEASE_URIBASE)/$@ -o $@

imports/%_import.owl: imports/%_import_pre.owl
	cp $< $@

#bio-chebi-merged.owl: bio-chebi.owl
#	$(OWLTOOLS) $(USECAT) $< --merge-imports-closure -o $@
#imports/chebi_import.owl: bio-chebi.owl imports/chebi_terms_combined.txt $(SRC)
#	$(ROBOT) extract -i $< -T imports/chebi_terms_combined.txt --method BOT -O $(BASE)/$@ annotate -V $(RELEASE_URIBASE)/$@ -o $@
CHEBIRELS = RO:0000057 RO:0002233 RO:0002234 RO:0002313 RO:0002233 RO:0002332 RO:0002340  RO:0002345 BFO:0000051 RO:0000087 GOCHEREL:0000000 GOCHEREL:0000001 GOCHEREL:0000002  GOCHEREL:0000003 GOCHEREL:0000004

#robot doesn't appear to work for making modules with GCIs; hence this contorted mechanism
# See: https://github.com/ontodev/robot/issues/175
seed.owl: $(SRC)
	$(OWLTOOLS) $(USECAT) $< --remove-imports-declarations -o $@
imports/chebi_import_nogci.owl: imports/chebi_terms_combined.txt $(SRC)
	$(ROBOT) extract -i bio-chebi.owl -T imports/chebi_terms_combined.txt --method BOT -O $(BASE)/$@ annotate -V $(RELEASE_URIBASE)/$@ -o $@
imports/chebi_import.owl: imports/chebi_import_nogci.owl seed.owl bio-chebi.owl
	$(OWLTOOLS) $(USECAT) seed.owl $< --merge-support-ontologies  bio-chebi.owl --add-imports-from-supports --extract-module -s $(OBO)/go/extensions/bio-chebi.owl -c  --make-subset-by-properties -n $(CHEBIRELS) // --remove-annotation-assertions -r -l --add-obo-shorthand-to-properties --set-ontology-id $(OBO)/go/$@ -o $@
.PRECIOUS: imports/chebi_import.owl

imports/%_import.obo: imports/%_import.owl
	$(OWLTOOLS) $(USECAT) $< -o -f obo --no-check $@.tmp && grep -v ^owl-axioms: $@.tmp > $@

# ----------------------------------------
# CACHEING EXTERNAL ONTOLOGIES
# ----------------------------------------

WGET_OUT = -O $@.tmp && cp $@.tmp $@ && touch $@

mirror:
	mkdir $@

# special case: download obo for speed for ncbitaxon.
# note also that here we only trigger new downloads if ncbitaxon_terms.txt is touched
mirror/ncbitaxon-download.owl: imports/ncbitaxon_terms.txt
	wget --no-check-certificate $(OBO)/ncbitaxon.obo $(WGET_OUT)
.PRECIOUS: mirror/ncbitaxon-download.owl
#mirror/ncbitaxon-download-enhanced.owl: mirror/ncbitaxon-download.owl
#	OWLTOOLS_MEMORY=12G owltools  $< --create-taxon-disjoint-over-in-taxon -s -r NCBITaxon:1 -m --set-ontology-id $OBO/zz -o
#.PRECIOUS: mirror/ncbitaxon-download-enhanced.owl

# special case: download obo for speed for ncbitaxon
mirror/pr-download.owl: $(SRC)
	wget --no-check-certificate $(OBO)/pr.obo $(WGET_OUT)
.PRECIOUS: mirror/pr-download.owl

# special case: download obo for speed for chebi
# See also: https://github.com/ebi-chebi/ChEBI/issues/3269
mirror/chebi-download.owl: $(SRC)
#	wget --no-check-certificate $(OBO)/chebi.obo $(WGET_OUT) && perl -pi -ne 's@BFO_@BFO:@' $@
	curl -L $(OBO)/chebi.obo  -o $@.tmp  && (test -f $@.tmp && mv $@.tmp $@ || echo unmodified) && perl -pi -ne 's@BFO_@BFO:@' $@
.PRECIOUS: mirror/chebi-download.owl

# special case: use ext for uberon
mirror/uberon-download.owl: $(SRC)
	wget --no-check-certificate $(OBO)/uberon/ext.owl $(WGET_OUT)
.PRECIOUS: mirror/uberon-download.owl

# special case: use basic for oba
mirror/oba-download.owl: $(SRC)
	wget --no-check-certificate $(OBO)/oba/subsets/oba-basic.obo -O $@.tmp && owltools $@.tmp --make-subset-by-properties -n // -o $@
.PRECIOUS: mirror/oba-download.owl

# special case: use basic for cl
mirror/cl-download.owl: $(SRC)
	wget --no-check-certificate $(OBO)/cl/cl-basic.obo $(WGET_OUT)
.PRECIOUS: mirror/cl-download.owl

# special case: the RO imports closure is unusual
# typically we use only the main ontology, not the imports.
# for RO we merge these, in order to get ro/core, bfo classes
# see https://github.com/geneontology/go-ontology/issues/12946
mirror/ro-download.owl: $(SRC)
	owltools $(OBO)/ro.owl --merge-imports-closure -o $@
.PRECIOUS: mirror/cl-download.owl

# general case: download remote OWL first
mirror/%-download.owl: $(SRC)
#	wget --no-check-certificate $(OBO)/$*.owl $(WGET_OUT)
	curl -L $(OBO)/$*.owl -o $@.tmp && (test -f $@.tmp && mv $@.tmp $@ || echo unmodified)
.PRECIOUS: mirror/%-download.owl

# bio-chebi.owl imports chebi and injects GCI axioms to conflate conjugate bases. See Hill et al
bio-chebi.owl: mirror/chebi-download.owl mirror/chebi.owl
	owltools $(USECAT) --create-biochebi -o $@ -c $<

# filter axioms
FILTER_EXTERNAL=--remove-axioms-about -d GO --remove-imports-declarations --remove-annotation-assertions -l -s -d -r --remove-axiom-annotations --remove-dangling-annotations --add-obo-shorthand-to-properties --set-ontology-id $(OBO)/$@

# todo: uberon currently has some chebi inferences
mirror/uberon.owl: mirror/uberon-download.owl
	$(OWLTOOLS) --use-catalog $< --remove-equivalent-to-nothing-axioms --remove-classes-in-idspace -d -s CHEBI --remove-classes-in-idspace -d -s NBO --remove-classes-in-idspace -d -s CARO --remove-classes-in-idspace -d -s CL  $(FILTER_EXTERNAL) -o $@
.PRECIOUS: mirror/uberon.owl

mirror/%.owl: mirror/%-download.owl
	$(OWLTOOLS) $< $(FILTER_EXTERNAL) -o $@
.PRECIOUS: mirror/%.owl

# ----------------------------------------
# MAPPINGS
# ----------------------------------------

generate-mappings: go-basic.obo
	$(OWLTOOLS) $< --external-mappings-files -o external2go --label-prefix GO: --externals EC KEGG MetaCyc Reactome RESID UM-BBD_enzymeID UM-BBD_pathwayID Wikipedia

# ----------------------------------------
# TAXON GROUPINGS
# ----------------------------------------

imports/go-taxon-groupings.owl: imports/go-taxon-groupings-edit.owl
	robot reason -i $< -r hermit -o $@

imports/go-taxon-groupings.obo: imports/go-taxon-groupings.owl
	robot convert -i $< -o $@.tmp.obo && grep -v ^owl-axioms $@.tmp.obo > $@

# ----------------------------------------
# VALIDATION
# ----------------------------------------

# these live in the ../sparql directory, and have suffix -violation.sparql
# adding the name here will make the violation check live
VCHECKS = equivalent-classes trailing-whitespace owldef-self-reference obsolete-reference

# run all violation checks
VARGS = $(foreach V,$(VCHECKS), $(SPARQLDIR)/$V-violation.sparql)
sparql_test: $(SRC)
	robot verify --input $< --queries $(VARGS) -O reports/

# use this to run tests on an individual basis
reports/%-violation.csv: $(SRC)
	robot query -i $< -s $(SPARQLDIR)/$*-violation.sparql $@

# ----------------------------------------
# REPORTS
# ----------------------------------------
# We use `robot query` to generate TSVs from SPARQL

REPORTS = owl-stats basic-report class-count-by-prefix edges labels xrefs obsoletes synonyms
REPORT_ARGS = $(foreach V,$(REPORTS),-s $(SPARQLDIR)/$V.sparql reports/$V.tsv)
all_reports: go.owl
	robot query -f tsv -i $< $(REPORT_ARGS)

#all_src_reports: $(patsubst %,reports/src-%.tsv,$(REPORTS))

reports/%.tsv: go.owl $(SPARQLDIR)/%.sparql $(REPORTDIR)
	robot -vv query -f tsv -i $< -s $(SPARQLDIR)/$*.sparql $@.tmp && mv $@.tmp $@
reports/src-%.tsv: go-edit.obo $(SPARQLDIR)/%.sparql $(REPORTDIR)
	robot query -f tsv -i $< -s $(SPARQLDIR)/$*.sparql $@.tmp && mv $@.tmp $@

# ----------------------------------------
# ADDING INFERENCES
# ----------------------------------------
# see: https://github.com/geneontology/go-ontology/issues/12315
# go inferences now go into a separate file.
# this is imported during an OE session (see oboedit.catalog)
# (for Protege all inferences are dynamic)

# first strip the go_inferences import; we need to do this
# because we cannot create a new ontology with the same name
# TEMPORARY: we should be able to remove this step soon
go-edit_noinf.obo: go-edit.obo
	grep -v '^import: http://purl.obolibrary.org/obo/go/imports/go_inferences.owl' $< > $@.tmp && mv $@.tmp $@

# use robot to place inferences in a new file.
# TBD: should relaxed axioms go here?
go_inferences.owl: go-edit_noinf.obo
	robot reason -i $< -r ELK -n true -a true -x true annotate -R -O $(OBO)/go/imports/go_inferences.owl -o $@.tmp.owl && mv $@.tmp.owl $@

# translation to obo.
# note that in order to get obo "!" comments into the file (to enhance readability), we must:
# (a) import the go ontology that contains the labels (b) save (c) strip out the import directive
# NOTE: in svn pipeline, this file was committed by jenkins job every time ontology is edited.
go_inferences.obo: go_inferences.owl
	owltools $< --use-catalog go-edit.obo --add-imports-from-supports -o -f obo $@.tmp && grep -v ^import: $@.tmp > $@
.PRECIOUS: go_inferences.obo

# TODO: change this once diff pipeline in place
#go_inferences.obo-previous: $(SRC)
#	wget --no-check-certificate http://geneontology.org/ontology/editors/go_inferences.obo -O $@

%-previous: $(SRC)
	wget --no-check-certificate https://build.berkeleybop.org/job/build-go-ontology-inferences/lastSuccessfulBuild/artifact/target/$* -O $@.tmp && mv $@.tmp $@ || echo "Fresh diffs" > $@

# get the difference between the current staged inferences file and the
# last one to be checked in.
go_inferences.diff: go_inferences.obo go_inferences.obo-previous
	echo getting log
	git log --date=iso $(SRC) | head -6 > $@
	echo >> $@
	echo "changes in entailments" >> $@
	echo >> $@
	echo diffing
	diff -u go_inferences.obo-previous go_inferences.obo >> $@ || echo differences found
.PRECIOUS: go_inferences.diff

# append latest diffs onto difflog
# NOTE: in svn pipeline, this file was committed by jenkins job every time ontology is edited.
go_inferences_difflog.txt: go_inferences.diff go_inferences_difflog.txt-previous
	cat $^ > $@.tmp && mv $@.tmp $@

# reasoner diffs
idiff: go_inferences_difflog.txt
	mkdir -p $(RELEASEDIR)/
	cp go_inferences_difflog.txt $(RELEASEDIR) && \
	cp go_inferences.diff $(RELEASEDIR) && \
	cp go_inferences.obo $(RELEASEDIR)
