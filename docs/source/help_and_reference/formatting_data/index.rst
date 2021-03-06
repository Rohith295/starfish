.. _formatting:

Converting Data to SpaceTx Format
=================================

We provide three types of tools to convert data into SpaceTx-Format. One is a `Bio-Formats`_ writer
which writes SpaceTx-Format experiments using the Bio-Formats converter. Bio-Formats can read a
variety of input formats, so might be a relatively simple approach for users familiar with those
tools.

.. _Bio-Formats: https://www.openmicroscopy.org/bio-formats/

Second, we provide a mechanism by which the user organizes the data as 2D tiles with a clearly defined filename schema, and a conversion tool.  There is :ref:`documentation and an example <format_structured_data>` for that mechanism.

If neither of these models fit, then we provide a :ref:`generalized mechanism <advanced_formatting>` where conversion is managed through a set of interfaces where the user provides python code responsible for obtaining the data corresponding to each 2D tile.  :ref:`Example formatters <data_conversion_examples>` for a variety of datasets are also available.  This same interface can also be used to :ref:`directly load data <tilefetcher_loader>`, although there may be performance implications in doing so.

.. _advanced_formatting:

Advanced data formatting
------------------------

Example Data Conversion
^^^^^^^^^^^^^^^^^^^^^^^

We provide an example for formatting an in-situ sequencing (ISS) experiment. These data were
generated by the Nilsson lab, and the analysis of the results can be found in their `publication`_.
In brief, this  experiment has 16 fields of view that measure 4 channels on 4 separate
imaging rounds.

.. _publication: https://www.nature.com/articles/nmeth.2563

We have
all the data from this experiment including the organization of the fields of view in physical
space, but we lack the exact physical coordinates. For the purpose of this tutorial, we
**fabricate** coordinates that are consistent with the ordering of the tiles and store those
coordinates in an auxiliary json file.

The fields of view for this dataset are organized in a grid and ordered as follows, with `1` being
the first position of the microscope on the tissue slide:

::

    [ 1   2   3   4  ]
    [ 8   7   6   5  ]
    [ 9   10  11  12 ]
    [ 16, 15, 14, 13 ]


To be consistent with this data, we've created a json file that contains fabricated coordinates for
each of the first two fields of view, which we will use for this tutorial. We set the origin
:code:`(0, 0, 0)` at the top left of the first field of view, so the coordinates for these files
are:

.. code-block:: json

    {
        "fov_000": {
            "xc": [0, 10],
            "yc": [0, 10],
            "zc": [0, 1]
        },
        "fov_001": {
            "xc": [10, 20],
            "yc": [0, 10],
            "zc": [0, 1]
        }
    }

Downloading Data
^^^^^^^^^^^^^^^^

Like all starfish example data, this experiment is hosted on Amazon Web Services. Once formatted,
experiments can be downloaded on-demand into starfish.

For the purposes of this vignette, we will format only two of the 16 fields of view. To download the
data, you can run the following commands:

.. code-block:: bash

    mkdir -p iss/raw
    aws s3 cp s3://spacetx.starfish.data.public/browse/raw/20180820/iss_breast/ iss/raw/ \
        --recursive \
        --exclude "*" \
        --include "slideA_1_*" \
        --include "slideA_2_*" \
        --include "fabricated_test_coordinates.json" \
        --no-sign-request
    ls iss/raw

This command should download 44 images:

- 2 fields of view
- 2 overview images: "dots" used to register, and DAPI, to localize nuclei
- 4 rounds, each containing:
- 4 channels (Cy 3 5, Cy 3, Cy 5, and FITC)
- DAPI nuclear stain

Formatting single-plane TIFF files in SpaceTx Format
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We provide some tools to take a directory of files like the one just downloaded and translate it
into starfish-formatted files. These objects are ``TileFetcher`` and ``FetchedTile``. In brief,
TileFetcher provides an interface to get the appropriate tile from a directory for a given
set of sptx-format metadata, like a specific z-plane, imaging round, and channel by decoding
the file naming conventions. ``FetchedTile`` exposes methods to extract data specific to each
tile to fill out the remainder of the metadata, such as the tile's shape and data. This particular
example is quite simple because the data are already stored as 2-D TIFF files, however there are
several other :ref:`examples <data_conversion_examples>` that convert more complex data into a
SpaceTx-Format :py:class:`Experiment`.


These are the abstract classes that must be subclassed for each set of naming conventions:

.. literalinclude:: /../../starfish/core/experiment/builder/providers.py
    :pyobject: FetchedTile

.. literalinclude:: /../../starfish/core/experiment/builder/providers.py
    :pyobject: TileFetcher

To create a formatter object for in-situ sequencing, we subclass the ``TileFetcher`` and
``FetchedTile`` by extending them with information about the experiment. When formatting
single-plane TIFF files, we expect that all metadata needed to construct the ``FieldOfView``
is embedded in the file names.

For the ISS experiment, the file names are structured as follows

.. code-block:: bash

    slideA_1_1st_Cy3 5.TIF

This corresponds to

.. code-block:: bash

    (experiment_name)_(field_of_view_number)_(imaging_round)_(channel_name).TIF

So, to construct a ``sptx-format`` ``FieldOfView`` we must adjust the basic TileFetcher object so
that it knows about the file name syntax.

That means implementing methods that return the shape, format, and an open file handle for a tile.
Here, we implement those methods, and add a cropping method as well, to mimic the way that ISS data
was processed when it was published.

.. literalinclude:: ../../../../examples/data_formatting/format_iss_breast_data.py
    :pyobject: IssCroppedBreastTile

This object, combined with a ``TileFetcher``, contains all the information that ``starfish`` needs
to parse a directory of files and create ``sptx-format`` compliant objects. Here, two tile fetchers
are needed. One parses the primary images, and another the auxiliary nuclei images that will be
used to seed the basin for segmentation.

.. literalinclude:: ../../../../examples/data_formatting/format_iss_breast_data.py
    :pyobject: ISSCroppedBreastPrimaryTileFetcher

.. literalinclude:: ../../../../examples/data_formatting/format_iss_breast_data.py
    :pyobject: ISSCroppedBreastAuxTileFetcher

Creating a Build Script
^^^^^^^^^^^^^^^^^^^^^^^

Next, we combine these objects with some information we already had about the experiments. On the
outset we stated that an ISS experiment has 4 imaging rounds and 4 channels, but only 1 z-plane.
These data fill out the ``primary_image_dimensions`` of the ``TileSet``. In addition, it was stated
that ISS has a single ``dots`` and ``nuclei`` image. In ``starfish``, auxiliary images are also
stored as ``TileSet`` objects even though often, as here, they have only 1 channel, round, and
z-plane.

We create a dictionary to hold each piece of information, and pass that to
``write_experiment_json``, a generic tool that accepts the objects we've aggregated above and
constructs TileSet objects:

.. literalinclude:: ../../../../examples/data_formatting/format_iss_breast_data.py
    :pyobject: format_data

Finally, we can run the script. We've packaged it up as an example in ``starfish``. It takes as
arguments the input directory (containing raw images), output directory (which will contain
formatted data) and the number of fields of view to extract from the raw directory.

.. code-block:: bash

    mkdir iss/formatted
    python3 examples/format_iss_breast_data.py \
        iss/raw/ \
        iss/formatted \
        2
    ls iss/formatted/*.json