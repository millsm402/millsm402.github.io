---
title: "Driving Marketing Impact with Data-Driven Personas"
excerpt: "Enabling personalized marketing outreach.<br/><img src='/files/port-pc-fig.jpg' style='width:75%;'>"
collection: portfolio
---

Here I partnered with the Marketing Team to develop data-driven personas that enhanced campaign precision and engagement. To do so I leveraged customer segmentation, behavioral analytics, and clustering techniques to translate raw data into actionable profiles that enable personalized outreach.

The project stood out by automating a human-in-the-loop clustering solution (Figure 1).

<!--  FIGURE: CLUSTERING SOLUTION PLOT  -->
Figure 1. Plot of decision criteria (CCC and Pseudo F) for determining the number of clusters. The ideal solution is when peaks overlap.
<img src="/files/port-clustersolutionplot.svg" alt="Cluster Solutions" style="max-width: 100%; height: auto;">

Cubic Clustering Criterion (CCC) and Pseudo F work together to identify an appropriate clustering solution. CCC is not native to Python, so here is the function that computes it. 

```python
# Function to calculate Cubic Clustering Criterion (CCC)
def calculate_ccc(data, clusters, n_clusters):
    n_samples = data.shape[0]
    n_features = data.shape[1]

    # Total sum of squares (TSS)
    grand_mean = np.mean(data, axis=0)
    tss = np.sum((data - grand_mean) ** 2)

    # Within-cluster sum of squares (WSS)
    wss = 0
    for cluster in range(n_clusters):
        cluster_points = data[clusters == cluster]
        cluster_mean = np.mean(cluster_points, axis=0)
        wss += np.sum((cluster_points - cluster_mean) ** 2)

    # Between-cluster sum of squares (BSS)
    bss = tss - wss

    # R-squared value
    r2 = bss / tss

    # Expected R-squared under null hypothesis (E_R2)
    e_r2 = 1 - (1 / (1 + (n_features + 2) / (n_clusters - 1) * (n_samples - n_clusters) / n_samples))

    # Variance of R-squared (Var_R2)
    var_r2 = 2 * (1 - e_r2) ** 2 / (n_samples - n_clusters)

    # Cubic Clustering Criterion (CCC)
    ccc = (r2 - e_r2) / np.sqrt(var_r2)
    
    return ccc, r2
