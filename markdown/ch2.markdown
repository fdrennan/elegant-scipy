## Quantile normalization with NumPy and SciPy

```python
%matplotlib inline
# Make all plots appear inline in the IPython notebook from now onwards

import matplotlib.pyplot as plt
plt.style.use('ggplot') # Use ggplot style graphs for something a little prettier
```

```python
# Need to check if all these import statements are actually required.
import numpy as np
from scipy import stats
import pandas as pd

import seaborn as sns
import itertools as it
```

### Get the data

```python
import numpy as np
import pandas as pd

# Import TCGA melanoma data
filename = 'data/counts.txt'
with open(filename, 'rt') as f:
    data_table = pd.read_csv(f, index_col=0) # Parse file with pandas

print(data_table.iloc[:5, :5])
```

```python
# Sample names
samples = list(data_table.columns)
```

```python
# Import gene lengths
filename = 'data/genes.csv'
with open(filename, 'rt') as f:
    gene_info = pd.read_csv(f, index_col=0) # Parse file with pandas, index by GeneSymbol
print(gene_info.iloc[:5, :5])
```

```python
#Subset gene info to match the count data
matched_index = data_table.index.intersection(gene_info.index)
print(gene_info.loc[matched_index].shape)
print(data_table.loc[matched_index].shape)
```

```python
# 1D ndarray containing the lengths of each gene
gene_lengths = np.asarray(gene_info.loc[matched_index]['GeneLength'],
                          dtype=int)
```

```python
# 2D ndarray containing expression counts for each gene in each individual
counts = np.asarray(data_table.loc[matched_index], dtype=int)
```

```python
def plot_col_density(data, xlabel=None):
    """For each column (individual) produce a density plot over all rows (genes).

    data : 2d nparray
    xlabel : x axis label
    """

    density_per_col = [stats.kde.gaussian_kde(col) for col in data.T] # Use gaussian smoothing to estimate the density
    x = np.linspace(np.min(data), np.max(data), 100)

    plt.figure()
    for density in density_per_col:
        plt.plot(x, density(x))
    plt.xlabel(xlabel)
    plt.show()


# Before normalization
log_counts = np.log(counts + 1)
plot_col_density(log_counts, xlabel="Log count distribution for each individual")
```

Given an expression matrix (microarray data, read counts, etc) of ngenes by nsamples, quantile normalization ensures all samples have the same spread of data (by construction). It involves:

(optionally) log-transforming the data
sorting all the data points column-wise
averaging the rows
replacing each column quantile with the quantile of the average column.
This can be done with NumPy and scipy easily and efficiently.
Assume we've read in the input matrix as X:

```python
import numpy as np
from scipy import stats

def quantile_norm(X):
    """Normalize the columns of X to each have the same distribution.

    Given an expression matrix (microarray data, read counts, etc) of M genes
    by N samples, quantile normalization ensures all samples have the same
    spread of data (by construction).

    The input data is log-transformed, then the data across each row are
    averaged to obtain an average column. Each column quantile is replaced
    with the corresponding quantile of the average column.
    The data is then transformed back to counts.

    Parameters
    ----------
    X : 2D array of float, shape (M, N)
        The input data, with M rows (genes/features) and N columns (samples).

    Returns
    -------
    Xn : 2D array of float, shape (M, N)
        The normalized data.
    """
    # log-transform the data
    logX = np.log2(X + 1)

    # compute the quantiles
    log_quantiles = np.mean(np.sort(logX, axis=0), axis=1)

    # compute the column-wise ranks; need to do a round-trip through list
    ranks = np.transpose([np.round(stats.rankdata(col)).astype(int) - 1
                        for col in X.T])
    # alternative: ranks = np.argsort(np.argsort(X, axis=0), axis=0)

    # index the quantiles for each rank with the ranks matrix
    logXn = log_quantiles[ranks]

    # convert the data back to counts (casting to int is optional)
    Xn = np.round(2**logXn - 1).astype(int)
    return(Xn)

# Example usage: quantile_norm(counts_lib_norm)
```

```python
# After normalization
log_quant_norm_counts = np.log(quantile_norm(counts)+1)

plot_col_density(log_quant_norm_counts, xlabel="Log count distribution for each individual")
```


## Principal Components Analysis

```python
from sklearn.decomposition import PCA

def PCA_plot(data):
    """Plot the first two principle components of the data

    Parameters
    ----------
    data : 2D ndarray of counts (assumed to be genes X samples)
    """

    #construct your NumPy array of data
    counts_transposed = np.array(data).T

    # Set up PCA, set number of components
    pca = PCA(n_components=2)

    # project data into PCA space
    pca_transformed = pca.fit_transform(counts_transposed)

    # Plot the first two principal components
    plt.scatter(x=pca_transformed[:,0], y=pca_transformed[:,1])
    plt.title("PCA")
    plt.xlabel("PC 1")
    plt.ylabel("PC 2")
    plt.show()
```

```python
PCA_plot(counts)
PCA_plot(quantile_norm(counts))
```

## Biclustering the counts data

Now that the data are normalized, we can cluster the genes (rows) and samples (columns) of the expression matrix.
Clustering the rows tells us which genes' expression values are linked, which is an indication that they work together in the process being studied.
Clustering the samples tells us which samples have similar gene expression profiles, which may indicate similar characteristics of the samples on other scales.

Because clustering can be an expensive operation, we will limit our analysis to the 1,500 genes that are most variable, since these will account for most of the correlation signal in either dimension.

```python
def most_variable_rows(data, n=1500):
    """Subset data to the n most variable rows

    In this case, we want the n most variable genes.

    Parameters
    ----------
    data : 2D array of float
        The data to be subset
    n : int, optional
        Number of rows to return.
    method : function, optional
        The function with which to compute variance. Must take an array
        of shape (nrows, ncols) and an axis parameter and return an
        array of shape (nrows,).
    """
    # compute variance along the columns axis
    rowvar = np.var(data, axis=1)
    # Get sorted indices (ascending order), take the last n
    sort_indices = np.argsort(rowvar)[-n:]
    # use as index for data
    variable_data = data[sort_indices, :]
    return variable_data
```

Next, we need a function to *bicluster* the data.
This means clustering along both the rows (to find out with genes are working together) and the columns (to find out which samples are similar).

Normally, you would use a sophisticated clustering algorithm from the [scikit-learn](http://scikit-learn.org) library for this.
In our case, we want to use hierarchical clustering for simplicity and ease of display.
The SciPy library happens to have a perfectly good hierarchical clustering module, though it requires a bit of wrangling to get your head around its interface.

As a reminder, hierarchical clustering is a method to group observations using sequential merging of clusters:
initially, every observation is its own cluster.
Then, the two nearest clusters are repeatedly merged, until every observation is in a single cluster.
This sequence of merges forms a *merge tree*.
By cutting the tree at a particular distance threshold, we can get a finer or coarser clustering of observations.

The `linkage` function in `scipy.cluster.hierarchy` performs a hierarchical clustering of the rows of a matrix, using a particular metric (for example, Euclidean distance, Manhattan distance, or others) and a particular linkage method, the distance between two clusters (for example, the average distance between all the observations in a pair of clusters).

It returns the merge tree as a "linkage matrix", which contains each merge operation along with the distance computed for the merge and the number of observations in the resulting cluster. From the `linkage` documentation:

> A cluster with an index less than $n$ corresponds to one of
> the $n$ original observations. The distance between
> clusters `Z[i, 0]` and `Z[i, 1]` is given by `Z[i, 2]`. The
> fourth value `Z[i, 3]` represents the number of original
> observations in the newly formed cluster.

Whew! So that's a lot of information, but let's dive right in and hopefully you'll get the hang of it rather quickly.
First, we define a function, `bicluster`, that clusters both the rows *and* the columns of a matrix:

```python
import matplotlib.pyplot as plt
from scipy.cluster.hierarchy import linkage, fcluster, dendrogram, leaves_list


def bicluster(data, linkage_method='average', distance_metric='correlation'):
    """Cluster the rows and the columns of a matrix.

    Parameters
    ----------
    data : 2D ndarray
        The input data to bicluster.
    linkage_method : string, optional
        Method to be passed to `linkage`.
    distance_metric : string, optional
        Distance metric to use for clustering. See the documentation
        for ``scipy.spatial.distance.pdist`` for valid metrics.

    Returns
    -------
    y_rows : linkage matrix
        The clustering of the rows of the input data.
    y_cols : linkage matrix
        The clustering of the cols of the input data.
    """
    y_rows = linkage(data, method=linkage_method, metric=distance_metric)
    y_cols = linkage(data.T, method=linkage_method, metric=distance_metric)
    return y_rows, y_cols
```

Simple: we just call `linkage` for the input matrix and also for the transpose of that matrix, in which columns become rows and rows become columns.

Next, we define a function to visualize the output of that clustering.
We are going to rearrange the rows an columns of the input data so that similar rows are together and similar columns are together.
And we are additionally going to show the merge tree for both rows and columns, displaying which observations belong together for each.

As a word of warning, there is a fair bit of hard-coding of parameters going on here.
This is difficult to avoid for plotting, where design is often a matter of eyeballing to find the correct proportions.

```python
def plot_bicluster(data, row_linkage, col_linkage,
                   row_nclusters=10, col_nclusters=3):
    """Perform a biclustering, plot a heatmap with dendrograms on each axis.

    Parameters
    ----------
    data : array of float, shape (M, N)
        The input data to bicluster.
    row_linkage : array, shape (M-1, 4)
        The linkage matrix for the rows of `data`.
    col_linkage : array, shape (N-1, 4)
        The linkage matrix for the columns of `data`.
    n_clusters_r, n_clusters_c : int, optional
        Number of clusters for rows and columns.
    """
    fig = plt.figure(figsize=(8, 8))

    # Compute and plot row-wise dendrogram
    # `add_axes` takes a "rectangle" input to add a subplot to a figure.
    # The figure is considered to have side-length 1 on each side, and its
    # bottom-left corner is at (0, 0).
    # The measurements passed to `add_axes` are the left, bottom, width, and
    # height of the subplot. Thus, to draw the left dendrogram (for the rows),
    # we create a rectangle whose bottom-left corner is at (0.09, 0.1), and
    # measuring 0.2 in width and 0.6 in height.
    ax1 = fig.add_axes([0.09, 0.1, 0.2, 0.6])
    # For a given number of clusters, we can obtain a cut of the linkage
    # tree by looking at the corresponding distance annotation in the linkage
    # matrix.
    threshold_r = (row_linkage[-row_nclusters, 2] +
                   row_linkage[-row_nclusters+1, 2]) / 2
    dendrogram(row_linkage, orientation='right', color_threshold=threshold_r)

    # Compute and plot column-wise dendogram
    # See notes above for explanation of parameters to `add_axes`
    ax2 = fig.add_axes([0.3, 0.71, 0.6, 0.2])
    threshold_c = (col_linkage[-col_nclusters, 2] +
                   col_linkage[-col_nclusters+1, 2]) / 2
    dendrogram(col_linkage, color_threshold=threshold_c)

    # Hide axes labels
    ax1.set_xticks([])
    ax1.set_yticks([])
    ax2.set_xticks([])
    ax2.set_yticks([])

    # Plot data heatmap
    ax = fig.add_axes([0.3, 0.1, 0.6, 0.6])

    # Sort data by the dendogram leaves
    idx_rows = leaves_list(row_linkage)
    data = data[idx_rows, :]
    idx_cols = leaves_list(col_linkage)
    data = data[:, idx_cols]

    im = ax.matshow(data, aspect='auto', origin='lower', cmap='YlGnBu_r')
    ax.set_xticks([])
    ax.set_yticks([])

    # Axis labels
    plt.xlabel('Samples')
    plt.ylabel('Genes', labelpad=125)

    # Plot legend
    axcolor = fig.add_axes([0.91, 0.1, 0.02, 0.6])
    plt.colorbar(im, cax=axcolor)

    # display the plot
    plt.show()
```

Now we apply these functions to our normalized counts matrix to display row and column clusterings.

```python
counts_log = np.log(counts + 1)
counts_var = most_variable_rows(counts_log, n=1500)
yr, yc = bicluster(counts_var)
plot_bicluster(counts_var, yr, yc)
```

We can see that the sample data naturally falls into at least 2 clusters.
Are these clusters meaningful?
To answer this, we can access the patient data, available from the [data repository](https://tcga-data.nci.nih.gov/docs/publications/skcm_2015/) for the paper.
After some preprocessing, we get the [patients table]() (LINK TO FINAL PATIENTS TABLE), which contains survival information for each patient.
We can then match these to the counts clusters, and understand whether the patients' gene expression can predict differences in their pathology.

```python
patients = pd.read_csv('data/patients.csv', index_col=0)
patients.head()
```

Now we need to draw *survival curves* for each group of patients defined by the clustering.
This is a plot of the fraction of a population that remains alive over a period of time.
Note that some data is *right-censored*, which means that in some cases, don't actually know when the patient died, or the patient might have died of causes unrelated to the melanoma.
We counts these patients as "alive" for the duration of the survival curve, but more sophisticated analyses might try to estimate their likely time of death.

To obtain a survival curve from survival times, we create a step function that decreases by $1/n$ at each step, where $n$ is the population size.
We then match that function against the non-censored survival times.

```python
def survival_distribution_function(lifetimes, right_censored=None):
    """Return the survival distribution function of a set of lifetimes.

    Parameters
    ----------
    lifetimes : array of float or int
        The observed lifetimes of a population. These must be non-
        -negative.
    right_censored : array of bool, same shape as `lifetimes`
        A value of `True` here indicates that this lifetime was not
        observed. Values of `np.nan` in `lifetimes` are also considered
        to be right-censored.

    Returns
    -------
    sorted_lifetimes : array of float
        The
    sdf : array of float
        Values starting at 1 and progressively decreasing, one level
        for each observation in `lifetimes`.

    Examples
    --------

    In this example, of a population of four, two die at time 1, a
    third dies at time 2, and a final individual dies at an unknown
    time. (Hence, ``np.nan``.)

    >>> lifetimes = np.array([2, 1, 1, np.nan])
    >>> survival_distribution_function(lifetimes)
    (array([ 0.,  1.,  1.,  2.]), array([ 1.  ,  0.75,  0.5 ,  0.25]))
    """
    n_obs = len(lifetimes)
    rc = np.isnan(lifetimes)
    if right_censored is not None:
        rc |= right_censored
    observed = lifetimes[~rc]
    xs = np.concatenate(([0], np.sort(observed)))
    ys = np.concatenate((np.arange(1, 0, -1/n_obs), [0]))
    ys = ys[:len(xs)]
    return xs, ys
```

Now that we can easily obtain survival curves from the survival data, we can plot them.
We write a function that groups the survival times by cluster identity and plots each group as a different line:

```python
def plot_cluster_survival_curves(clusters, sample_names, patients,
                                 censor=True):
    """Plot the survival data from a set of sample clusters.

    Parameters
    ----------
    clusters : array of int or categorical pd.Series
        The cluster identity of each sample, encoded as a simple int
        or as a pandas categorical variable.
    sample_names : list of string
        The name corresponding to each sample. Must be the same length
        as `clusters`.
    patients : pandas.DataFrame
        The DataFrame containing survival information for each patient.
        The indices of this DataFrame must correspond to the
        `sample_names`. Samples not represented in this list will be
        ignored.
    censor : bool, optional
        If `True`, use `patients['melanoma-dead']` to right-censor the
        survival data.
    """
    plt.figure()
    if type(clusters) == np.ndarray:
        cluster_ids = np.unique(clusters)
        cluster_names = ['cluster {}'.format(i) for i in cluster_ids]
    elif type(clusters) == pd.Series:
        cluster_ids = clusters.cat.categories
        cluster_names = list(cluster_ids)
    n_clusters = len(cluster_ids)
    for c in cluster_ids:
        clust_samples = np.flatnonzero(clusters == c)
        # discard patients not present in survival data
        clust_samples = [sample_names[i] for i in clust_samples
                         if sample_names[i] in patients.index]
        patient_cluster = patients.loc[clust_samples]
        survival_times = np.array(patient_cluster['melanoma-survival-time'])
        if censor:
            censored = ~np.array(patient_cluster['melanoma-dead']).astype(bool)
        else:
            censored = None
        stimes, sfracs = survival_distribution_function(survival_times,
                                                        censored)
        plt.plot(stimes / 365, sfracs)

    plt.xlabel('survival time (years)')
    plt.ylabel('fraction alive')
    plt.legend(cluster_names)
```

Now we can use the `fcluster` function to obtain cluster identities for the samples (columns of the counts data), and plot each survival curve separately.
The `fcluster` function takes a linkage matrix, as returned by `linkage`, and a threshold, and returns cluster identities.
It's difficult to know a-priori what the threshold should be, but we can obtain the appropriate threshold for a fixed number of clusters by checking the distances in the linkage matrix.

```python
n_clusters = 3
threshold_distance = (yc[-n_clusters, 2] + yc[-n_clusters+1, 2]) / 2
clusters = fcluster(yc, threshold_distance, 'distance')

plot_cluster_survival_curves(clusters, data_table.columns, patients)
```

The clustering of gene expression profiles has identified a higher-risk subtype of melanoma, which constitutes the majority of patients.
This is indeed only the latest study to show such a result, with others identifying subtypes of leukemia (blood cancer), gut cancer, and more.
Although the above clustering technique is quite fragile, there are other ways to explore this dataset and similar ones that are more robust.

**Exercise:** We leave you the exercise of implementing the approach described in the paper:

1. Take bootstrap samples (random choice with replacement) of the genes used to cluster the samples;
2. For each sample, produce a hierarchical clustering;
3. In a `(n_samples, n_samples)`-shaped matrix, store the number of times a sample pair appears together in a bootstrapped clustering.
4. Perform a hierarchical clustering on the resulting matrix.

This identifies groups of samples that frequently occur together in clusterings, regardless of the genes chosen.
Thus, these samples can be considered to robustly cluster together.

*Hint: use `np.random.choice` with `replacement=True` to create bootstrap samples of row indices.*