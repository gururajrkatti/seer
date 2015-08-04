# seer
Sequence element enrichment analysis

Installation
==============
First clone the repository

    git clone https://github.com/johnlees/seer

Also include dlib, if necessary

    git clone --recursive https://github.com/johnlees/seer

Currently tested on Linux only, installation should proceed as

    cd src
    make
    make install

Dependencies
--------------
seer currently depends on

- boost <http://www.boost.org/>
- gzstream <http://www.cs.unc.edu/Research/compgeom/gzstream/>
- armadillo <http://arma.sourceforge.net/>
- dlib <http://www.dlib.net/>
- HDF5 <https://www.hdfgroup.org/HDF5/>

You will also require

- gcc >4.9 or equivalent
- gcc libstdc++ >4.9

Brief installation instructions

**boost**

Best installed with your distribution's package manager, and you should use the c++11 version if possible.

For a manual installation, see <http://www.boost.org/doc/libs/1_57_0/more/getting_started/unix-variants.html> for details on how to use ./b2 to install. I reccommend that you create a user-config.jam file in the boost root which modifies the gcc compilation:

    using gcc:
      : std11
      : g++
      : <cxxflags>-std=c++11

Then run

    ./bootstrap.sh
    ./b2 install toolset=gcc-std11

**gzstream**

Download and unpack. Change into directory

    make

**armadillo**

Download and unpack. Change into directory

    cmake .
    make
    make install

**dlib**

If not installed use the above git clone command to include with the
repository. Otherwise unpack header files to $(PREFIX)/include

**HDF5**

Best installed with your distribution's package manager. Otherwise use
a binary from <https://www.hdfgroup.org/HDF5/release/obtain5.html>, or
if you wish to compile from source

    gunzip < hdf5-X.Y.Z.tar.gz | tar xf -
    cd hdf5-X.Y.Z
    ./configure --prefix=/usr/local/hdf5 <more configure_flags>
    make
    make check
    make install
    make check-install

Usage
=============
The wrapper script is panpipes.py, in the top level directory. Main programs are installed into bin.

kmds
--------------
First, filter kmers and create a matrix representing population structure with kmds

    ./kmds -k dsm_input.txt.gz --pheno metadata.pheno -o filtered

To spread this process out, run the following command on each dsm file

    ./kmds -k dsm_input.txt.gz --pheno metadata.pheno --no_mds --size 10000

Then use

    ./kmds --mds_concat subsampled_matrices.txt -o all_structure --threads 16

seer
--------------
Type ./seer with no options to get brief usage, or for a full option listing

    ./seer -h

Basic usage is as follows

    ./seer -k dsm_input.txt.gz --pheno metadata.pheno > significant_kmers.txt

To use the kmds output, and increase execution speed

    ./seer -k filtered.gz --pheno metadata.pheno --struct filtered.dsm --no_filtering --threads 4

Troubleshooting
=============

General
-------------
**When I run the programs I get an error about a missing GLIBCXX**

    seer: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.17' not found (required by seer)
    seer: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by seer)

Make sure you've got the gcc libstdc++ v4.9 or higher available. The
default on your OS may be a much lower version than this

kmds
-------------
**This takes ages to run**
First, split up your input files into say 16 pieces, then subsample each
one in its own process using --no_mds and --size.

Then list these output files in a text file, and input using
--mds_concat. Using 16 threads this took about 30 hour and 53Gb on our test
system of 3 000 samples, using 1% of the kmer.

Choose the number of kmers to subsample carefully, lower and this will run
faster and require less memory. A size of 1% of the number of kmers is
appropriate, but as low as 0.1% should work.

seer
-------------
**The p-values are slightly different from what I expect**
For very small p-values (when W > 5) an upper bound is calculated rather
than the exact p-value for robustness and computational speed. This
bound is accurate to +/- W^(-2) %

If you need the exact value, checkout the erfc branch which uses
arbitrary precision floats and doesn't rely on this bound.

**I get p-values of 0**
p-value is too small to represent as a float (<10E-308). At this point
it is a bit meaningless to assign a value, but essentially the data
exactly fits the regression.

Biologically, either the effect size is enormous, or more likely the
phenotype is monophyletic (see below).

**I get the error message: 'inv_sympd(): matrix appears to be singular'**
The optimiser may not have converged, check previous error messages and
see the note about this below.

If it has, it's likely the p-value is too small to calculate. You can
approximate this p-value as 0 if you wish (the program default). If
you're getting lots of messages like this your phenotype is probably
perfectly associated with a clade, or small number of clades.

Check whether this is the case using a tool such as [phylocanvas](http://phylocanvas.org/).
If your phenotype is monophyletic then you won't have good resolution on
which sequence elements may be related to it. See the paper for more
explanation on this

**I get the error message: 'kmer convergence error: The objective function generated non-finite outputs'**
The mds structure is probably poorly scaled compared to the kmers. kmers
presence and absence is coded as 1 and 0 respectively.

kmds produced a rectangular matrix of mds values, where each row is
a sample, and each column is a decreasing dimension. Each dimension is
scaled to values in the range [-1,1].

These matrices are stored in hdf5 format, so you can use these tools to
inspect them, and if necessary rescale them. Most convenient is the
R package [rhdf5](http://www.bioconductor.org/packages/release/bioc/html/rhdf5.html).
