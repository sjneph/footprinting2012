## Footprint Detection ##
> Shane Neph


Overview
=========
The core program that implements the DNaseI [footprinting description] from: An expansive human regulatory lexicon encoded in transcription factor footprints, Nature 489, 83–90 (2012).

Given a set of sequencing tag counts (integers) at each base of any region, this program creates an unthresholded list of candidate footprints.


Build
=====
// requires g++ version 4.7 or newer  
make -C src/


Program Options:
================
```
detect-cache
	[--flankmin <bases>  = 6]
	[--flankmax <bases>  = 12]
	[--centermin <bases> = 6]
	[--centermax <bases> = 100]
	[--maxthold <value>  = 10]
	File-Full-O-Integers
```

The footprint occupancy score (FOS) of a candidate footprint is defined as:  
FOS = (C+1)/L + (C+1)/R,  
where C is the average number of tags over the central/core region of a potential footprint, while L (R) is the average tag level found in the left (right) flanking region.

--flankmin  
--flankmax set the min/max number of flanking bases over which to find the max mean value for L or R  

--centermin  
--centermax set the min/max number of bases over which to find the min mean value for C  

--maxthold should be ignored so that the program uses the default value of 10  


Input
=====
This program accepts a file full of integers that represent the number of cleavage events assigned to each base.  That is, each base receives an integer that shows the number of uniquely-mapping sequencing tags with 5' ends mapping to that position.  For those bases with none, they receive a zero.  Since cleavage occurs in between two nucleotides, choose the 5'-most of these to represent each cleavage event.  It is not strictly necessary to use only uniquely-mapping tags though it is common in practice.

You may use a dash ('-') to denote that input comes from stdin.

Regions of interest can be broken up into subsequences by file.  Breaking up your data by chromosome is the most natural partition when running things in parallel.


Results
=======
The output of this program consists of unthresholded candidate footprints.  The output is deterministic so if you run everything again with the same inputs, you will receive the same output.  The file format has 8 columns.

1. The leftmost position of the left-flanking region
2. The start of the central/core potential footprint (1 bp beyond the end of left-flanking region)
3. The start of the right-flanking region (1 bp beyond the end of the core region)
4. The end of the right-flanking region (1 bp beyond the end of the right-flanking region)
5. The FOS
6. The mean tag level of the left-flanking region (L)
7. The mean tag level of the core region (C)
8. The mean tag level of the right-flanking region (R)

All candidate footprints' core regions are disjoint and they do not abutt.  Each output region has been optimized over the input parameter settings.  You will next want to threshold results further, including the removal of candidate footprints with too many unmapped bases in their core regions.  Another common filter is to ignore results outside of DNaseI FDR 1% hotspot regions.

Note that the program does not know about chromosomes.  Further, it reads in and interprets the first integer as belonging to absolute position zero.  If you feed it something that does not start at base zero, you need to adjust the output using the first input base's offset.

Even if you paste on chromosome information to the beginning of the output, be careful as the output is not in [sorted BED order].

Typically, one thresholds the potential footprints based upon some metric that utilizes the FOS, rearranges columns 2&3 to 1&2, pastes on appropriate chromosome information as the first field, and then sorts to obtain the final result.  Using this procedure, you end up with 0-based [start,end) footprint calls where columns 2 and 3 hold the core footprint start and end positions.


Portability
===========
This program runs fine on Linux systems, Mac OS X, and BSD systems.  It is written in standard C++, so it should compile and run on Windows systems though a different build manager would need to replace our simple makefile.  The makefile hardcodes g++ as the compiler.  Change CC in the makfile (for example, to clang++) along with build flags as you see fit.


Tips
====
One method of sticking zeroes in for bases that have no per-base number of cuts [using bedops] looks like:
```
  bedops -c -L <bases-with-tag-counts> \
    | awk '{ for (i=$2; i<$3; ++i) { print $1,i,i+1,".",0; } }' \
    | bedops -u - <bases-with-tag-counts> \
    | cut -f5
```

Note that ```<bases-with-tag-counts>``` must be [properly sorted], and the output of this command sequence can be piped directly into the _detect-cache_ program.


Performance and scalability
===========================
We regularly run this program on deeply-sequenced data using a compute cluster.  We break the genome up by chromosome and submit each to the cluster.  When using this method, you can expect full results in less than one hour with a genome roughly the size of the human genome.  One could restrict inputs to less than a whole genome (for example, restrict to 1% FDR DNaseI hotspots) in order to speed up computations considerably.  The tradeoff is a significantly larger amount of bookkeeping to create inputs and to glue the final results together.

The program can use a bit of main memory and we recommend 2G or more RAM.  Surprisingly, feeding the program all zeroes gives the worst case memory performance (and it will likely use up all of your main memory).  That is something that I plan to address in the future.  Consequently, for now, having sequencing tags over more bases in the genome improves memory performance.  Note that we have never had any memory issues in practice when using real data sets with 30 million or more uniquely-mapping sequencing tags.


[footprinting description]: http://www.nature.com/nature/journal/v489/n7414/extref/nature11212-s1.pdf
[sorted BED order]: https://bedops.readthedocs.org/en/latest/content/reference/file-management/sorting/sort-bed.html
[properly sorted]: https://bedops.readthedocs.org/en/latest/content/reference/file-management/sorting/sort-bed.html
[using bedops]: https://bedops.readthedocs.org/en/latest/content/reference/set-operations/bedops.html
