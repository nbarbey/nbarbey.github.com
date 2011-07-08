--- 
title: Sumatra introduction
layout: post
---

[Sumatra](http://neuralensemble.org/trac/sumatra/) is a very nice tool
to do
[reproducible research](http://en.wikipedia.org/wiki/Reproducibility#Reproducible_research)
or simply to manage numerical experiments in an efficient way.

There is already a very well written
[documentation](http://packages.python.org/Sumatra/) with a
[getting started section](http://packages.python.org/Sumatra/getting_started.html).
It assumes that the reader is familiar with
[revision control](http://en.wikipedia.org/wiki/Revision_control)
software. This is also very well covered on the web (see for instance
the
[mercurial quick start page](http://mercurial.selenic.com/quickstart/)).

But I would like to show how simple it really is to use sumatra to
people unfamiliar with this way of doing numerical experiments.  I
will illustrate this with a script taken from
[tamasis-map](http://pchanial.github.com/tamasis-map/) which performs
map-making for the
[Herschel Space Observatory](http://en.wikipedia.org/wiki/Herschel_Space_Observatory)
and in particular the [PACS instrument](http://pacs.mpe.mpg.de/).

A simple script to perform map-making looks like this : 

{% highlight python %}

#!/usr/bin/env python

import os
import tamasis as tm

# PACS data filename
filename = 'frames_blue.fits'
# load metadata
obs = tm.PacsObservation(filename=filename)
# load data
tod = obs.get_tod(flatfielding=False)
# define instrument model
projection   = tm.Projection(obs, resolution=3.2, oversampling=False,
                            npixels_per_sample=6)
masking_tod  = tm.Masking(tod.mask)
model = masking_tod * projection
# perform map-making
map_rls = tm.mapper_rls(tod, model, hyper=1., tol=1.e-4)
# save result
map_rls.save("map.fits")

{% endhighlight %}

This script consists in a few simple functions parametrized by a small
number of keywords. It takes a [FITS file](http://fits.gsfc.nasa.gov/)
as input and output another FITS file. The script can be executed from
the command line but no argument can be passed from the command line.

This is not what Sumatra expects. Sumatra expects a script which
accepts a configuration file which defines the parameters of the
numerical experiment. The configuration file should be passed from the
command line to the script and parsed.

So we should rewrite our map-making script in this manner. We can
separate it in two parts. A part parsing the configuration file.
This can be done for instance with the ConfigParser package.

{% highlight python %}

#!/usr/bin/env python

options = "hvf:o:"
long_options = ["help", "verbose", "filenames=", "output="]

def main():
    import os, sys, getopt, ConfigParser, time
    # parse command line arguments
    try:
        opts, args = getopt.getopt(sys.argv[1:], options, long_options)
    except getopt.GetoptError, err:
        print(str(err))
        usage()
        sys.exit(2)

    # default
    verbose = False

    # parse options
    for o, a in opts:
        if o in ("-h", "--help"):
            usage()
            sys.exit()
        if o in ("-v", "--verbose"):
            verbose = True

    # read config file
    if len(args) == 0:
        print("Error: config filename argument is mandatory.\n")
        usage()
        sys.exit(2)
    config_file = args[0]
    config = ConfigParser.RawConfigParser()
    config.read(config_file)
    keywords = dict()
    sections = config.sections()
    sections.remove("main")
    # parse config
    for section in sections:
        keywords[section] = dict()
        for option in config.options(section):
            get = config.get
            # recast to bool, int or float if needed
            if section == "PacsObservation":
                if option == "reject_bad_line":
                    get = config.getboolean
                if option == "fine_sampling_factor":
                    get = config.getint
                if option in ("active_fraction",
                              "delay",
                              "calblock_extension_time"):
                    get = config.getfloat
            if section == "get_tod":
                if option in ("flatfielding",
                              "substraction_mean",
                              "raw"):
                    get = config.getboolean
            if section == "Projection":
                if option in ("oversampling",
                              "packed"):
                    get = config.getboolean
                if option == "npixels_per_sample":
                    get = config.getint
                if option == "resolution":
                    get = config.getfloat
            if section == "mapper_rls":
                if option in ("verbose",
                              "criterion"):
                    get = config.getboolean
                if option == "maxiter":
                    get = config.getint
                if option in ("hyper",
                              "tol"):
                    get = config.getfloat
            # store option using the appropriate get to recast option.
            keywords[section][option] = get(section, option)
    # special case for the main section
    data_file_list = config.get("main", "filenames").split(",")
    # if filenames argument is passed, override config file value.
    for o, a in opts:
        if o in ("-f", "--filenames"):
            data_file_list = a.split(", ")
    data_file_list = [w.lstrip().rstrip() for w in data_file_list]
    # append date string to the output file to distinguish results.
    date = time.strftime("%y%m%d_%H%M%S", time.gmtime())
    filename = data_file_list[0].split(os.sep)[-1]
    # remove extension
    fname = ".".join(filename.split(".")[:-1])
    # store results into the Data subdirectory as expected by sumatra
    output_file = "Data/map" + fname + '_' + date + '.fits'
    # if output argument is passed, override config file value.
    for o, a in opts:
        if o in ("-o", "--output"):
            output_file = a
    # run tamasis mapper
    pipeline_rls(data_file_list, output_file, keywords, verbose=verbose)

# to call from command line
if __name__ == "__main__":
    main()

{% endhighlight %}

This is a long code. But the longest part is used to define the data
type of the arguments. The idea here is that a section of the
configuration file correspond to a step of the map-making script.  A
dictionary storing the parameters of each step is filled from the
values in the corresponding section.

Now the map-making script can be rewritten as a function accepting a
dictionary of dictionaries as follows :

{% highlight python %}

def pipeline_rls(filenames, output_file, keywords, verbose=False):
    """
    Perform regularized least-square inversion of a simple model (tod
    mask and projection). Processing steps are as follows :
    
        - define PacsObservation instance
        - get Time Ordered Data (get_tod)
        - define projection
        - perform inversion on model
        - save file

    Arguments
    ---------
    filenames: list of strings
        List of data filenames.
    output_file : string
        Name of the output fits file.
    keywords: dict
        Dictionary containing options for all steps as dictionary.
    verbose: boolean (default False)
        Set verbosity.

    Returns
    -------
    Returns nothing. Save result as a fits file.
    """
    from scipy.sparse.linalg import cgs
    import tamasis as tm
    # verbosity
    keywords["mapper_rls"]["verbose"] = verbose
    # define observation
    obs_keys = keywords.get("PacsObservation", {})
    obs = tm.PacsObservation(filenames, **obs_keys)
    # get data
    tod_keys = keywords.get("get_tod", {})
    tod = obs.get_tod(**tod_keys)
    # define projector
    proj_keys = keywords.get("Projection", {})
    projection = tm.Projection(obs, **proj_keys)
    # define mask
    masking_tod  = tm.Masking(tod.mask)
    # define full model
    model = masking_tod * projection
    # perform map-making inversion
    mapper_keys = keywords.get("mapper_rls", {})
    map_rls = tm.mapper_rls(tod, model, solver=cgs, **mapper_keys)
    # save
    map_rls.save(output_file)


{% endhighlight %}

It is also easy to add some documentation to the script :

{% highlight python %}

def usage():
    print(__usage__)

__usage__ = """Usage: pacs_rls [options] [config_file]

Use tamasis regularized least-square map-making routine
using the configuration [filename]

[config_file] is the name of the configuration file. This file
contains arguments to the various processing steps of the
map-maker. For an exemple, see the file tamasis_rls.cfg in the
tamasis/pacs/src/ directory.

Options:
  -h, --help        Show this help message and exit
  -v, --verbose     Print status messages to std output.
  -f, --filenames   Overrides filenames config file value.
  -o, --output      Overrides output default value.
"""

{% endhighlight %}

The configuration file would look like this :

{% highlight ini%}

[main]
filenames=frames_blue.fits

[PacsObservation]
fine_sampling_factor=1
policy_bad_detector=mask
reject_bad_line=False
policy_inscan=keep
policy_turnaround=keep
policy_other=remove
policy_invalid=mask
active_fraction=0
delay=0.0

[get_tod]
unit=Jy/detector
flatfielding=True
subtraction_mean=True
raw=False
masks=activated

[Projection]
#resolution=3.2
#npixels_per_sample=6
oversampling=True
packed=False

[mapper_rls]
hyper=1.0
tol=1.e-5
maxiter=300

{% endhighlight %}

With this code, you can do from the command line :

{% highlight bash %}
pacs_rls.py pacs_rls.cfg
{% endhighlight %}

Or using sumatra :

{% highlight bash %}
smt run --executable=python --main=pacs_rls.py pacs_rls.cfg
{% endhighlight %}

Note that the files you use with Sumatra should be under revision
control, with Mercurial for instance.
