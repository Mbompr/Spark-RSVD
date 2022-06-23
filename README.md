[![Build Status](https://api.travis-ci.org/criteo/Spark-RSVD.svg?branch=master)](https://travis-ci.org/criteo/Spark-RSVD)

# Spark-RSVD
Spark-RSVD is a lib to compute approximate SVD decomposition of large sparse matrices (up to 100 million rows and columns) using an iterative algorithm for speed and efficiency.

The iterative algorithm is based on a random initial starting point, hence its name of Randomized SVD algorithm. It is described in \[[1](#references)\]. A tree reduce algorithm is used for fast QR decomposition of tall and skinny matrices \[[2](#references)\].

## Inputs and outputs

Spark-RSVD expects a sparse matrix of the type `BlockMatrix` as defined [here](src/main/scala/com/criteo/rsvd/BlockMatrix.scala#L250). To help creating such a matrix, a helper function (`FromMatrixEntries` [code](src/main/scala/com/criteo/rsvd/BlockMatrix.scala#L78)) is provided. It accepts an RDD of `MatrixEntry`, as defined by [MLLib](https://spark.apache.org/docs/2.1.0/api/java/org/apache/spark/mllib/linalg/distributed/MatrixEntry.html), which is a simple case class to indicate non-zero values in coordinate format. 

Spark-RSVD also needs some other parameters as inputs which are explained in the [parameters](#parameters) section below.

Spark-RSVD outputs the left singular vectors, the singular values and the right singular vectors (if needed). The singular values are stored as a [Breeze](https://github.com/scalanlp/breeze) `DenseVector[Double]`. The singular vectors are stored as a `SkinnyBlockMatrix` defined [here](src/main/scala/com/criteo/rsvd/SkinnyBlockMatrix.scala#L99).

## Parameters

Spark-RSVD needs some parameters which are collected together in a [case class](src/main/scala/com/criteo/rsvd/RSVD.scala#L24) called `RSVDConfig`. They are mostly self-explanatory, but here is a description of their meaning:

- `embeddingDim`: `Int`. Number of the approximate singular vectors and singular values to be computed
- `oversample`: `Int`. Number of surplus vectors which are needed to increase the precision of the singular vectors. See \[[1](#references)\] for more explanation.
- `powerIter`: `Int`. Number of iterations of the iterative algorithm. Only a couple are needed to get to a high precision (again, see \[[1](#references)\])
- `seed`: `Int`. Seed for the initialization of the random basis. Using always the same value should lead to repeatable results (though Spark may have unrepeatable results due to the varying order of execution of some operations)
- `blockSize`: `Int`. Size of the blocks used in `BlockMatrix` and `SkinnyBlockMatrix`. See the section [data format](#data-format) for more explanations.
- `partitionWidthInBlocks`: `Int`. Width of the partitions of the `BlockMatrix` in number of blocks. The `SkinnyBlockMatrix` is also partitioned vertically with the same number of blocks for consistency during the matrix-vector multiplication. See the section [data format](#data-format) for more explanations.
- `partitionHeightInBlocks`: `Int`. Height of the partitions of the `BlockMatrix` in number of blocks. See the section [data format](#data-format) for more explanations.
- `computeLeftSingularVectors` and `computeRightSingularVectors`: `Boolean`. Indicates whether the left singular vectors and the right singular vectors should be computed.

### Sensible configuration

An important configuration part of Spark-RSVD library is the tuning how much memory per partition you should use.
Using too much memory per Spark partition will likely results in tasks failing due to Out Of Memory errors,
however we provide several configuration parameters to tune how the data should be distributed:
- `blockSize`
- `partitionWidthInBlocks`
- `partitionHeightInBlocks`

The most memory-intensive operation that occurs within Spark-RSVD computation is a distributed multiplication of a Sparse Matrix with a Dense "tall-and-skinny" matrix, so if we can ensure that this step works, the whole pipeline will likely succeed.
Given the way we have chosen to distribute this operation, a given Spark partition will receive during this step the following inputs:
- a number of "blocks" from the Sparse Matrix, precisely `partitionWidthInBlocks` * `partitionHeightInBlock` blocks. These blocks are square blocks of `blockSize` size.
- a number of "blocks" from the tall-and-skinny Matrix, precisely `partitionWidthInBlocks` blocks. These blocks have `blockSize` rows and (`embeddingDim`+`oversample`) columns

Let's compute the overall amount of data that a partition will receive for a given configuration.
For this example we are choosing `blockSize` = 50000, `partitionWidthInBlocks` = 35, `partitionHeightInBlocks` = 10, `embeddingDim` = 100, `oversample` = 30 and we are assuming a `density` of 10<sup>-5</sup> for the sparse matrix.
This means that a partition will receive the following data during the multiplication step:
- partitionWidthInBlocks * blockSize * blockSize * partitionHeightInBlocks * 8 * 10<sup>-5</sup> = 66 MB of data coming from the sparse matrix
- partitionWidthInBlocks * blockSize * (embeddingDim+oversample) * 8 = 1.7 GB of data coming from the tall-and-skinny matrix.
Given these results, this means that you should make sure that you have roughly 2GB of memory available per Spark task if you want to run Spark-RSVD with this configuration.
Obviously, having less memory available for Spark tasks mean that you should tune your Spark-RSVD configuration accordingly

## Data format

The sparse matrix given to the library should be stored in [coordinate format](https://en.wikipedia.org/wiki/Sparse_matrix#Coordinate_list_(COO)). It expects an RDD of `MatrixEntry` (from Spark MLLib `org.apache.spark.mllib.linalg.distributed`), which stores the row and column indices as `Long` and the value as `Double`. See example below for further reference.

## Examples

### Scala example

Here is an example that will compute a 100-dimension embedding on a 200K * 200K matrix

```Scala
import com.criteo.rsvd._

// Create spark context
val sc: SparkContext = new SparkContext(...)

// Create RSVD configuration
val config = RSVDConfig(
  embeddingDim = 100,
  oversample = 30,
  powerIter = 1,
  seed = 0,
  blockSize = 50000,
  partitionWidthInBlocks = 35,
  partitionHeightInBlocks = 10,
  computeLeftSingularVectors = true,
  computeRightSingularVectors = true
)

val matHeight = 200000 // 200K
val matWidth = 200000 // 200K
val numNonZeroEntries = 400000 // 400K

// Generate a sparse random matrix as an input (doesn't have to be symmetric)
val randomMatrixEntries = sc.parallelize(0 until numNonZeroEntries).map {
  idx =>
    val random = new Random(42 + idx)
    MatrixEntry(random.nextInt(matHeight), //row index
                random.nextInt(matWidth), //column index
                random.nextGaussian()) //entry value
}

val matrixToDecompose = BlockMatrix.fromMatrixEntries(randomMatrixEntries,
                                           matHeight = matHeight,
                                           matWidth = matWidth,
                                           config.blockSize,
                                           config.partitionHeightInBlocks,
                                           config.partitionWidthInBlocks)

val RsvdResults(leftSingularVectors, singularValues, rightSingularVectors) =
  RSVD.run(matrixToDecompose, config, sc)

// Print the top 100 (embeddingDim=100) singular values in decreasing order:
println(singularValues.toString())

// Fetch the left-singular vectors to driver, which will be a 200K x 100 matrix.
// This is available because we set config.computeLeftSingularVectors = true.
val leftSingularOnDriver = leftSingularVectors.get.toLocalMatrix

// Fetch the right-singular vectors to driver, which will be a 200K x 100 matrix.
// This is available because we set config.computeRightSingularVectors = true.
val rightSingularOnDriver = rightSingularVectors.get.toLocalMatrix
```

### Pyspark example

This example is extracted from [example.py](python/example.py) where you will 
be able to see the PySpark setup and the helpers we suggest you to use
(`RSVDConfig` and `run_rvsd`).

```python
config = RSVDConfig(
    embedding_dim=100,
    oversample=30,
    power_iter=1,
    seed=0,
    block_size=1_000,
    partition_width_in_blocks=35,
    partition_height_in_blocks=10,
    compute_left_singular_vectors=True,
    compute_right_singular_vectors=True,
)

mat_height = 10_000
mat_width = 10_000
num_non_zero_entries = 20_000
density = num_non_zero_entries / (mat_height * mat_width)

random_matrix = sps.random(mat_height, mat_width, density)
random_matrix_pdf = pd.DataFrame({"row": random_matrix.row, "col": random_matrix.col, "data": random_matrix.data})
random_matrix_df = ss.createDataFrame(random_matrix_pdf)

left_embeddings_df, singular_values_array, right_embeddings_df = run_rvsd(
    random_matrix_df, row_index_column="row", column_index_column="col", value_column="data", config=config
)
```

## Installation

### Scala installation

To use, simply add the following dependency to your gradle build file

```groovy
compile "com.criteo:rsvd:1.1"
```

or the following dependency to your project's POM file

```xml
<dependency>
  <groupId>com.criteo</groupId>
  <artifactId>rsvd</artifactId>
  <version>1.1</version>
  <scope>compile</scope>
</dependency>
```

### Python installation

You must first download the jar and then add it to your spark config when you launch it.
See [example.py](python/example.py) for a standalone toy example in a local Spark context.

## References:

\[1\] Halko, N., Martinsson, P. G., & Tropp, J. A. (2011). Finding structure with randomness: Probabilistic algorithms for constructing approximate matrix decompositions. SIAM review, 53(2), 217-288.

\[2\] Constantine, P. G., & Gleich, D. F. (2011, June). Tall and skinny QR factorizations in MapReduce architectures. In Proceedings of the second international workshop on MapReduce and its applications (pp. 43-50). ACM.

## Authors of the initial commit (in alphabetical order):
- Aloïs Bissuel
- Vincent Grosbois
- Ivan Lobov

## License

This project is licensed under the Apache 2.0 license.

## Copyright

Copyright © Criteo, 2018.
