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

Cubic Clustering Criterion (CCC) and Pseudo F work together to identify an appropriate clustering solution. CCC is not native to Python, so I wrote a function (calculate_ccc) to compute it. 

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
```

The calculate_ccc function can then be called within the clustering function below, kmeans_clustering. This function is the driver — it runs the clustering, calls the first function for CCC, plots results, and returns insights. 

```python
def kmeans_clustering(maxiter, var_list, data):
    fitstat = pd.DataFrame(columns=['Cluster', 'F', 'CCC', 'R2'])

    for k in range(2, maxiter + 1):
        # Standardizing the data
        scaler = StandardScaler()
        scaled_data = scaler.fit_transform(data[var_list])

        # K-means clustering
        kmeans = KMeans(n_clusters=k, max_iter=100, n_init=10, random_state=42) #adding random_state=42 for stability
        clusters = kmeans.fit_predict(scaled_data)                    
        data['Cluster'] = clusters  

        # Calculate fit statistics
        pseudo_f_stat = calinski_harabasz_score(scaled_data, clusters)
        ccc, r2 = calculate_ccc(scaled_data, clusters, k)

        # Combining fit statistics
        fitstat = pd.concat([fitstat, pd.DataFrame({'Cluster': [k], 'F': [pseudo_f_stat], 'CCC': [ccc], 'R2': [r2]})])
        
        # Dimension reduction and scatter plot (using PCA for visualization purposes)
        pca = PCA(n_components=2)
        pca_result = pca.fit_transform(scaled_data)
        data['can1'] = pca_result[:, 0]
        data['can2'] = pca_result[:, 1]

        plt.figure()
        plt.scatter(data['can1'], data['can2'], c=clusters, cmap='viridis')
        plt.title(f'K-Means Clustering with {k} Clusters')
        plt.xlabel('Canonical Variable 1')
        plt.ylabel('Canonical Variable 2')
        plt.colorbar(label='Cluster')
        plt.show()

    # Plotting CCC and Pseudo F against the number of clusters
    plt.figure()
    ax1 = plt.gca()
    ax2 = ax1.twinx()
    fitstat.plot(kind='line', x='Cluster', y='CCC', ax=ax1, color='blue', legend=True)
    fitstat.plot(kind='line', x='Cluster', y='F', ax=ax2, color='red', legend=True)
    ax1.set_xlabel('Number of Clusters')
    ax1.set_ylabel('CCC (higher is better)', color='blue')
    ax2.set_ylabel('Pseudo F (higher is better)', color='red')
    plt.xticks(range(2, 16, 1))
    plt.title('CCC and Pseudo F against Number of Clusters')
    plt.show()
```