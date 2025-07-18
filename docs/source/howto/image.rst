The pyAFQ Image API
~~~~~~~~~~~~~~~~~~~
In the process of tractometry, it is sometimes necessary to apply images to
the data. By default, pyAFQ can automatically generate these images from the
DWI data. However, pyAFQ also has a system for users to specify which image to
use, called the Image API. Specifying an image, in general, is not as simple as
just providing a path to a file, because each subject and each session will
have a different image. That is why we developed pyAFQ's Image API. 

Currently, there are three different images that pyAFQ uses for tractometry:

#. The brain image. This is used to mask the dwi data and throw out any signal
   that is outside of the brain. It is typically applied before fitting ODF
   models. By default, it is calculated using :class:`AFQ.definitions.image.B0Image`.

#. The tractography seed image. This image determines where tractography is
   seeded. If it is floating point, the image is thresholded interally after
   interpolation using the seed_threshold parameter. This is recommended.
   However, the seed image can aslo be a binary image. By default, the
   seed image is :class:`AFQ.definitions.image.ScalarImage` (best_scalar) where best_scalar is chosen by the API
   based on valid scalars (typically "dti_fa"). 

#. The tractography stop image. This image determines where tractography stops.
   If it is floating point, the image is thresholded interally after
   interpolation using the stop_threshold parameter. This is recommended.
   However, the stop image can aslo be a binary image. By default, the
   stop image is :class:`AFQ.definitions.image.ScalarImage` (best_scalar) where best_scalar is chosen by the API
   based on valid scalars (typically "dti_fa"). 

In AFQ/definitions/image.py, there are several image classes one can use to specify images.
As a user, one should initialize image classes and pass them to the AFQ.api objects,
or write out the initialization as a string inside of one's configuration file
for use with the CLI.

- :class:`AFQ.definitions.image.ImageFile`: The simplest image class is :class:`AFQ.definitions.image.ImageFile`. If the image you want to use
  is already generated, use this class. It is initialized using BIDS filters,
  which pyAFQ will use to find images for each subject in each session.

- :class:`AFQ.definitions.image.FullImage`: The :class:`AFQ.definitions.image.FullImage` class is used if one does not want to mask at all.
  This image is True everywhere.

- :class:`AFQ.definitions.image.RoiImage`: All ROIs are "logically or'd" together in subject space to create
  this image. This is useful to provide as the seed image. In segmentation,
  pyAFQ retains only streamlines that pass through the ROIs. So, for
  efficiency, one can choose to only seed in the ROIs in the first place.

- :class:`AFQ.definitions.image.B0Image`: This image uses dipy's median_otsu on the subject's b0 to generate
  an image. This is the default brain image.

- :class:`AFQ.definitions.image.LabelledImageFile`: This image is similar to :class:`AFQ.definitions.image.ImageFile`. It is also initialized
  using BIDS filters but instead expects to find a labelled segmentation file.
  In the initialization, the user also provides inclusive and exclusive
  labels. These specify which labels to include and which labels to exclude
  in the image. If both inclusive and exclusive labels are set, the combine
  parameter is used to specify how to combine the images generated by the
  inclusive and exclusive labels. A common use of this class would be to pass
  BIDS filters to some segmentation file, and also set exclusive_labels = [0].
  This will mark all labels that are not 0 as a part of the image, and can
  be used as a brain image.

- :class:`AFQ.definitions.image.ThresholdedImageFile`: This image is similar to :class:`AFQ.definitions.image.ImageFile`. It allows the user to
  also provide a lower and upper bound. The bounds threshold the data from
  the file. Note that for the tractography seed and stop images, thresholding
  should typically be done using the seed_threshold and stop_threshold
  parameters, not using an already thresholded image.

- :class:`AFQ.definitions.image.ScalarImage`: This image is initialized only by specifying a scalar. Use this
  image if you want an image to be based on a scalar that pyAFQ already
  calculates, like "dti_fa" or "dti_md".

- :class:`AFQ.definitions.image.ThresholdedScalarImage`: This image is similar to :class:`AFQ.definitions.image.ScalarImage`. It allows the user to
  also provide a lower and upper bound. The bounds threshold the scalar data.
  Note that for the tractography seed and stop images, thresholding
  should typically be done using the seed_threshold and stop_threshold
  parameters, not using an already thresholded image.

- :class:`AFQ.definitions.image.PFTImage`: A image for specifying the segmentations used in PFT. Should only
  be used as a stop image. It's three arguments are three other images, which
  specify the three segmentations: white matter, gray matter, and
  corticospinal fluid.

- :class:`AFQ.definitions.image.CombinedImage`: This class can be used to combine the other images. It takes
  a list of images and allows the user to specify whether they should be
  combined using a logical "and" or "or".

Here is an example of using the :class:`AFQ.definitions.image.RoiImage` and :class:`AFQ.definitions.image.LabelledImageFile` on the HCP
data with the AFQ.api objects::

    from AFQ.data.fetch import fetch_hcp
    from AFQ.api.group import GroupAFQ
    import AFQ.definitions.image as afm

    # Download a subject to the AWS Batch machine from s3
    _, hcp_bids = fetch_hcp(
        [1],
        profile_name=False,
        study=f"HCP_1200")

    # make 500,000 seeds randomly distributed in the ROIs
    tracking_params = {
        "seed_image": afm.RoiImage(),
        "n_seeds": 500000,
        "random_seeds": True}

    # use segmentation file from HCP to get a brain image,
    # where everything not labelled 0 is considered a part of the brain
    brain_image_definition = afm.LabelledImageFile(
        suffix='seg', {'scope': 'dmriprep'}, exclusive_labels=[0])

    # define the api GroupAFQ object
    myafq = GroupAFQ(
        hcp_bids,
        brain_image_definition=brain_image_definition,
        tracking_params=tracking_params)

    # export_all runs the entire pipeline and creates many useful derivates
    myafq.export_all()
