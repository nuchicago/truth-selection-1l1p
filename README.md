Truth-Based 1l1p Selection
==========================
A. Mastbaum <mastbaum@uchicago.edu>, 2017/08/13

Updated: G. Putnam <grayputnam@uchicago.edu> on 2018/01/22

Loop through MCTracks and MCShowers, building a list of tracks on
showers using truth variables which can be configured to be distorted,
then selected based on that list.  

Selection code is built as a standalone executable which uses Gallery as
a library. Scans through Monte Carlo events and produces an output root
file which stores both truth level information and the distorted
information. Usage information below.

Code to generate histograms and covariance plots from this output is
build as a gallery-framework module and is run using a python script.
Further code to generate plots live in a number of python scripts. Usage
information below.

Tools
-----
### TSSelection: Truth-Based Selection ###
This module is the main truth-based selection code. It scans through Monte
Carlo events to extract MCTracks and MCShowers, distorts the truth
information based on configuration, applies selection cuts 
(e.g. kinetic energy), and classifies events as 1eip and 1muip. 
The exact signal regions (0p/1p/Np/Ntrack) can also be configured.
The selected events are written into an output analysis tree.

Note on Selection: I have recently been updating the way that selection
writes data to the output TTree's, and I have added in some more
configuration options. I haven't been able to run the code recently
however, and so these most recent features are untested, and may be a
little buggy. I'll remove this section when I've had a chance to test
stuff.

### RunSelection: Running the TSSelection Code ###

RunSelection serves as the entry point for the selection code. Parses
arguments from the command line and uses those arguments to configure
the selection. See the code for what command line arguments are used.
Note that the parsing is somewhat fragile.

**Selection Usage:**

To compile the code, make sure that you are using the proper make file
(should be building `truth_selection`) and run `make`.

Then, to run the selection in the command line, call:

    $ ./truth_selection [input_files] -o output_file --args

Where `[input_files]` is a list of space-separated input root files,
`output_file` is the name is the output root file to be written with the
output analysis trees, and `--args` are any other command line 
arguments you with to use.

# Command Line Arguments #
To be written.

**Selection Usage on the Grid:**

It is also possible to run the selection code on the grid using the
python file `submitJobs.py`, which internally uses `jobsub_submit` to
run grid jobs. `submitJobs.py` takes in a file containing a list of 
input file names (`-i`), the inclusive index of the first file to be 
used as input in that list (`-f`), and the exclusive index of the last 
file to be used as input (`-l`). The number of input files per job can
be configured using `-n`. `submitJobs.py` also takes as input a
number of arguments that can be passed to `truth_selection`. It can also
be run in debug mode (`-d`), where it will not run the generated shell
script, but rather just write it out to a file.

`submitJobs.py` assumes that the `truth_selection` binary is in the
directory where it is run, and it assumes that any `config.root` file is
there is as well if the argument for using a config file is set.

Minimal Example call:

    $ python submitJobs.py -i file_with_file_list -f 0 -l $length_of_file_list

The file list can be generated by `scripts/utils/make_file_list.py`,
given an input glob, and the POT of the file list can be calculated by
`scripts/utils/run_pot.py`. 

### TSCovariance: Covariance Matrix Generator ###
This module takes analysis trees with selected event samples and Monte Carlo
event weights, and computes a covariance matrix for a user-specified subset
of weights. The output is a weighted event histogram, a graph with systematic
uncertainty bands, and the covariance and correlation matrices.

**Usage:**

    $ python scripts/cov.py input.root

Here, the input is an analysis tree ROOT file, i.e. the output of
`TSSelection`.

The user should modify `cov.py` to select an appropriate combinations of
weights.



### TSPDFGen: PDF Generator ###

(No longer used)

The PDF Generator produces probability distributions used by `TSSelection` for
track and particle identification. It constructs a 2D histogram in dE/dx vs.
residual range for tracks, and a 1D histogram of dE/dx for showers, based on
the Monte Carlo truth information in the events in the input dataset.

**Usage:**

    $ python scripts/make_pdfs.py "input*.root"

The input is a set of art ROOT files.

This will create a file `ana_out.root` containing histograms named like
`htrackdedx_<PDG>` for track-like PDG codes, and `hshowerdedx_<PDG>` for
shower-like PDGs.

A set of PDFs (which is used by `TSSelection` is provided in
`data/dedx_pdfs.root`. To update the PDFs for the selection, run `TSPDFGen`
and replace that file.

### TSUtil: Utilities ###
This file contains common utilities used by the truth-based selection modules,
which are in namespace `tsutil`.

### Scripts ###
The `scripts` directory contains:

* Python scripts used to run each module (see the Tools section)
* Utilities in `scripts/util`
  * File handling and MC POT counting
* Analysis scripts in `scripts/analysis`

See header blocks at the top of each script for more information, including
usage.

Workflow
--------
The workflow begins with raw art ROOT files, produced by LArSoft (uboonecode).
In practice we use two Monte Carlo samples: one fully inclusive BNB neutrino
sample with MC cosmic ray overlays, and one with only nue events (to
improve the statistics for nues, since they constitute only a small fraction
of the BNB inclusive sample). Additionally, there are MC samples produced for
various signal models.

In general, MC weighting functions that assign weights based on variations in
the systematics parameters are not run on the production MC files, so we
must run them manually using grid-based LArSoft jobs that run the EventWeight
module (and can also drop most data products, since this analysis only looks
at MC truth). In this way we obtain copies of the MC datasets with only MC
information, imbued with a set of systematics weights attached to each event.

The selection script (see TSSelection above) operates on a list of these
weighted art ROOT files, applying the truth-based selection. Typically this
would be run three times -- for the electron, inclusive, and signal samples --
and produce three output files. The electron and inclusive samples can now be
added together, using `scripts/util/add.C`. This (in contrast to `hadd`) takes
CC nue and nuebar events from the electron sample and *everything else* from
the inclusive sample. Splitting in this way simplifies the reweighting, as
we combine samples representing different POT exposures. After the addition,
we have two ROOT files -- a combined 1e1p + 1mu1p background file, and a signal
file -- containing simple analysis ROOT trees.

When running on the grid, the output will be scattered in a number of
different root files. The helper scripts `scripts/utils/run_merge.py`
and `scripts/utils/run_merge_many_selections.py` can be used to merge
these into a single root file.

We can now run `scripts/cov.py` on the background ROOT file. This script will
iterate through the chosen weights (the set of which comprise a systematics
"universe") and produce a number of weighted spectra with and without
systematic error bands, plus the covariance and correlation matrices. The
script allows the user to toggle on and off different combinations of
parameters, so that one can produce a covariance for e.g. the flux
uncertainties alone, one particular GENIE uncertainty, or everything together.
The `plotcov.py` script is provided for convenience, to draw these plots into
reasonable-looking PDF files.

The `scripts/analysis` directory provides some basic analysis code which
operates on either the analysis tree files, or the histograms output from
`cov.py` The main script, `chi2.py` reads in the latter and can make a
variety of plots, and compute a chi^2 significance for observing the signal
prediction under (background-only) null hypothesis using an analytic
approximation or a Monte Carlo approach, which represents the final product
of this analysis chain.

There are two main normalization factors that these last steps requre:
the POT and the efficiency. The POT can be calculated using
`scripts/utils/run_pot.py` and is normalized in TSCovariance. The POT 
that you want to use when calculating significance/generating plots 
can then be set in `scripts/analysis/chi2.py`. In addition, the efficiency 
can be caluclated from counters in the output TTRee's from TSSelection 
(which are correctly added up by the mergeing scripts) and is normalized 
in `scripts/analysis/chi2.py` to be a number reasonable for a real analysis
(30% by default).
