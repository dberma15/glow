==============================================
Genome-wide Association Study Regression Tests
==============================================

Glow contains functions for performing simple regression analyses used in
genome-wide association studies (GWAS).

Linear regression
=================

``linear_regression_gwas`` performs a linear regression association test optimized for performance
in a GWAS setting.

Example
-------

.. code-block:: py

  from pyspark.ml.linalg import DenseMatrix
  import pyspark.sql.functions as fx
  import numpy as np

  # Read in VCF file
  df = spark.read.format('vcf')\
    .option("splitToBiallelic", True)\
    .load('/databricks-datasets/genomics/1kg-vcfs')\
    .limit(10).cache()

  # Generate random phenotypes and an intercept-only covariate matrix
  n_samples = df.select(fx.size('genotypes')).first()[0]
  covariates = DenseMatrix(n_samples, 1, np.ones(n_samples))
  phenotypes = np.random.random(n_samples).tolist()
  covariates_and_phenotypes = spark.createDataFrame([[covariates, phenotypes.tolist()]],
    ['covariates', 'phenotypes'])

  # Run linear regression test
  display(df.crossJoin(covariates_and_phenotypes).selectExpr(
    'contigName',
    'start'
    'names'
    # genotype_states returns the number of alt alleles for each sample
    'expand_struct(linear_regression_gwas(genotype_states(genotypes), phenotypes, covariates))'))


Parameters
----------

.. list-table::
  :header-rows: 1

  * - Name
    - Type
    - Details
  * - ``genotypes``
    - ``array<double>`` (or numeric type that can be cast to ``double``)
    - A numeric representation of the genotype for each sample at a given site, for example the
      result of the ``genotype_states`` function. This parameter can vary for each row in the dataset.
  * - ``covariates``
    - ``spark.ml`` ``Matrix``
    - A matrix containing the covariates to use in the linear regression model. Each row in the
      matrix represents observations for a sample. The indexing must match that of the ``genotypes``
      array that is, the 0th row in the covariate matrix should correspond to the same sample as the
      0th element in the ``genotypes`` array. This matrix must be constant for each row in the
      dataset. If desired, you must explicitly include an intercept covariate in this matrix.
  * - ``phenotypes``
    - ``array<double>`` (or numeric type that can be cast to ``double``)
    - A numeric representation of the phenotype for each sample. This parameter may vary for each
      row in the dataset. The indexing of this array must match the ``genotypes`` and
      ``covariates`` parameters.

Return
------

The function returns a struct with the following fields. The computation of each value matches the
`lm R package <https://www.rdocumentation.org/packages/stats/versions/3.6.1/topics/lm>`_.

.. list-table::
  :header-rows: 1

  * - Name
    - Type
    - Details
  * - ``beta``
    - ``double``
    - The fit effect coefficient of the ``genotypes`` parameter.
  * - ``standardError``
    - ``double``
    - The standard error of ``beta``.
  * - ``pValue``
    - ``double``
    - The P-value of the t-statistic for ``beta``.

Implementation details
----------------------

The linear regression model is fit using the QR decomposition. For performance, the QR decomposition
of the covariate matrix is computed once and reused for each (``genotypes``, ``phenotypes``) pair.

Logistic regression
===================

``logistic_regression_gwas`` performs a logistic regression hypothesis test optimized for performance
in a GWAS setting.

Example
-------

.. code-block:: py

  display(df.crossJoin(covariates_and_phenotypes).selectExpr(
    'contigName',
    'start'
    'names'
    'expand_struct(logistic_regression_gwas(genotype_states(genotypes), phenotypes, covariates, \'LRT\'))'))


Parameters
----------

The parameters for the logistic regression test are largely the same as those for linear regression. The primary
differences are that the ``phenotypes`` values should be in the set ``[0,1]`` and that there is one additional
parameter ``test`` to specify the hypothesis test method.

.. list-table::
  :header-rows: 1

  * - Name
    - Type
    - Details
  * - ``genotypes``
    - ``array<double>`` (or numeric type that can be cast to ``double``)
    - A numeric representation of the genotype for each sample at a given site, for example the
      result of the ``genotype_states`` function. This parameter can vary for each row in the dataset.
  * - ``covariates``
    - ``spark.ml`` ``Matrix``
    - A matrix containing the covariates to use in the logistic regression model. Each row in the
      matrix represents observations for a sample. The indexing must match that of the ``genotypes``
      array that is, the 0th row in the covariate matrix should correspond to the same sample as the
      0th element in the ``genotypes`` array. This matrix must be constant for each row in the
      dataset. If desired, you must explicitly include an intercept covariate in this matrix.
  * - ``phenotypes``
    - ``array<double>`` (or numeric type that can be cast to ``double``)
    - A numeric representation of the phenotype for each sample. This parameter may vary for each
      row in the dataset. The indexing of this array must match the ``genotypes`` and
      ``covariates`` parameters.
  * - ``test``
    - ``string``
    - The hypothesis test method to use. Currently the only supported method is the likelihood ratio test (``LRT``).

Return
------

The function returns a struct with the following fields. The computation of each value matches the
`glm R package <https://www.rdocumentation.org/packages/stats/versions/3.6.1/topics/glm>`_.

.. list-table::
  :header-rows: 1

  * - Name
    - Type
    - Details
  * - ``beta``
    - ``double``
    - Log-odds associated with the ``genotypes`` parameter, ``NaN`` if the fit failed.
  * - ``oddsRatio``
    - ``double``
    - Odds ratio associated with the ``genotypes`` parameter, ``NaN`` if the fit failed..
  * - ``waldConfidenceInterval``
    - ``array<double>``
    - Wald 95% confidence interval of the odds ratio, ``NaN`` s if the fit failed.
  * - ``pValue``
    - ``double``
    - p-value for the specified ``test``, ``NaN`` if the fit failed.

Implementation details
----------------------

The logistic regression null model and fully-specified model are fit using Newton iterations. For performance, the null
model is computed once for each ``phenotype`` and used as a prior for each (``genotypes``, ``phenotypes``) pair.

Example notebook and blog post
------------------------------

A detailed example and explanation of a GWAS workflow is available `here <https://databricks.com/blog/2019/09/20/engineering-population-scale-genome-wide-association-studies-with-apache-spark-delta-lake-and-mlflow.html>`_.

.. notebook:: .. tertiary/gwas.html
  :title: GWAS notebook