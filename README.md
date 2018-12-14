# Implementation of "Consensus-Driven Propagation in Massive Unlabeled Data for Face Recognition" (CDP)

### Notice
<span style="color:red">Code is in continuous updating, please pull before execution.</span>

### Paper

Xiaohang Zhan, Ziwei Liu, Junjie Yan, Dahua Lin, Chen Change Loy, ["Consensus-Driven Propagation in Massive Unlabeled Data for Face Recognition"](http://openaccess.thecvf.com/content_ECCV_2018/papers/Xiaohang_Zhan_Consensus-Driven_Propagation_in_ECCV_2018_paper.pdf), ECCV 2018

Project Page:
[link](http://mmlab.ie.cuhk.edu.hk/projects/CDP/)

### Dependency
Please use Python3, as we cannot guarantee its compatibility with python2. The version of PyTorch we use is 0.3.1. Other depencencies:

    ```
    pip install nmslib
    ```

### Usage
0. Clone the repo.

    ```shell
    git clone git@github.com:XiaohangZhan/cdp.git
    cd cdp
    ```

#### Using ready-made data

1. Download the data [here](https://drive.google.com/open?id=1Fs8oN1JiGJRC93TkfDV-PaCeXQz-Htea) to the repo root, and uncompress it.

    ```shell
    tar -xf data.tar.gz
    ```

2. Make sure the structure looks like the following:

    ```shell
    cdp/data/
    cdp/data/labeled/emore_l200k/
    cdp/data/unlabeled/emore_u200k/
    # ... other directories and files ...
    ```

3. Run CDP

    * Single model case:

        ```shell
        python -u main.py --config experiments/emore_u200k_single/config.yaml
        ```

    * Multi-model voting case (committee size: 4):

        ```shell
        python -u main.py --config experiments/emore_u200k_cmt4/config.yaml
        ```

    * Multi-model mediator case (committee size: 4):

        ```shell
        # edit `experiments/emore_u200k_cmt4/config.yaml` as following:
        # strategy: mediator
        python -u main.py --config experiments/emore_u200k_cmt4/config.yaml
        ```

4. Collect the results

    Take `Multi-model mediator case` for example, the results are stored in `experiments/emore_u200k_cmt4/output/k15_mediator_111_th0.9915/sz600_step0.05/meta.txt`. The order is the same as that in `data/unlabeled/emore_u200k/list.txt`. The samples labeled as `-1` are discarded by CDP. You may assign them with new unique labels if you must use them.

#### Using your own data

1. Create your data directory, e.g. `mydata`

    ```shell
    mkdir data/unlabeled/mydata
    ```

2. Prepare your data list as `list.txt` and copy it to the directory.

3. (optional) If you want to evaluate the performance on your data, prepare the meta file as `meta.txt` and copy it to the directory.

4. Prepare your feature files. Extract face features corresponding to the `list.txt` with your trained face models, and save it as binary files via `feature.tofile("xxx.bin")` in numpy. The features should satisfy `Cosine Similarity` condition. Finally link/copy them to `data/unlabeled/mydata/features/`. We recommand renaming the feature files using model names, e.g., `resnet18.bin`. CDP works for single model case, but we recommend you to use multiple models (i.e., preparing multiple feature files extracted from different models) with `mediator` for better results.

5. The structure should look like:

    ```shell
    cdp/data/unlabeled/mydata/
    cdp/data/unlabeled/mydata/list.txt
    cdp/data/unlabeled/mydata/meta.txt (optional)
    cdp/data/unlabeled/mydata/features/
    cdp/data/unlabeled/mydata/features/*.bin
    ```

    (You do not need to prepare knn files.)

6. Prepare the config file. Please refer to the examples in `experiments/`

    ```shell
    mkdir experiments/myexp
    cp experiments/emore_u200k_cmt4/config.yaml experiments/myexp/
    # edit experiments/myexp/config.yaml to fit your case.
    # you may need to change `base`, `committee`, `data_name`, etc.
    ```

7. Tips for paramters adjusting
    * Modify `threshold` to obtain roughly closed `precision` and `recall` to achieve higher `fscore`.
    * Higher threshold results in higher precision and lower recall.
    * Larger `max_sz` results in lower precision and higher recall.

### Run baselines

* We also implement several baseline clustering methods including: KMeans, MiniBatch-KMeans, Spectral, Hierarchical Agglomerative Clustering (HAC), FastHAC, DBSCAN, HDBSCAN, KNN DBSCAN, Approximate Rank-Order.

    ```shell
    sh run_baselines.sh # results stored in `baseline_output/`
    ```

### Evaluation Results

1. data: emore_u200k (This data set is not the one in the paper which cannot be released, but the relative results are similar.)
    * images number: 200K;
    * identity number: 2,577 (original annotation)

    * **baselines**

    | method                                | #clusters | prec, recall, fscore | total time |
    |---------------------------------------|-----------|----------------------|------------|
    | * kmeans (ncluster=2577)              | 2577      | 94.24, 74.89, 83.45  | 618.1s     |
    | * MiniBatchKMeans (ncluster=2577)     | 2577      | 89.98, 87.86, 88.91  | 122.8s     |
    | * Spectral (ncluster=2577)            | 2577      | 97.42, 97.05, 97.24  | 12.1h      |
    | * HAC (ncluster=2577, knn=30)         | 2577      | 97.74, 88.02, 92.62  | 5.65h      |
    | FastHAC (distance=0.7, method=single) | 46767     | 99.79, 53.18, 69.38  | 1.66h      |
    | DBSCAN (eps=0.75, nim_samples=10)     | 2330      | 0.6775, 73.77, 1.343 | 6.87h      |
    | HDBSCAN (min_samples=10)              | 2516      | 2.214, 79.87, 4.308  | 4.87h      |
    | KNN DBSCAN (knn=80, min_samples=10)   | 2494      | 1.358, 78.99, 2.669  | 60.5s      |
    | ApproxRankOrder (knn=20, th=10)       | 85150     | 52.96, 16.93, 25.66  | 86.4s      |

    note: Methods marked * are reported with their theoretical upper bound results, since they need number of clusters as input. We use the values from the ground truth to obtain the results. For each method, we adjust the parameters to achieve the best performance.

    * **CDP**

    | k  | strategy | committee | setting         | prec, recall, fscore | knn time | cluster time | total time |
    |----|----------|-----------|-----------------|----------------------|----------|--------------|------------|
    | 15 | vote     | 0         | accept0_th0.66  | 89.35, 88.98, 89.16  | 14.8s    | 7.7s         | 22.5s      |
    | 15 | vote     | 4         | accept4_th0.61  | 93.36, 92.91, 93.13  | 78.7s    | 6.0s         | 84.7s      |
    | 15 | mediator | 4         | 110_th0.9938    | 94.06, 92.45, 93.25  | 78.7s    | 77.7s        | 156.4s     |
    | 15 | mediator | 4         | 111_th0.9925    | 96.66, 94.93, 95.79  | 78.7s    | 100.2s       | 178.9s     |
    
    note: for mediator, `110` means using `relationship` and `affinity`; `111` means using `relationship`, `affinity` and `structure`.

2. data: emore_u1.4m
    * images number: 1.4M;
    * identity number: 21,433 (original annotation)

    * **CDP**
    
    | k  | strategy | committee | setting        | prec, recall, fscore | knn time | cluster time | total time |
    |----|----------|-----------|----------------|----------------------|----------|--------------|------------|
    | 15 | vote     | 0         | accept0_th0.68 | 89.49, 81.25, 85.17  | 187.5s   | 47.7s        | 235.2s     |
    | 15 | vote     | 4         | accept4_th0.62 | 90.63, 87.32, 88.95  | 967.0s   | 44.3s        | 1011.3s    |
    | 15 | mediator | 4         | 110_th0.99     | 93.67, 84.43, 88.81  | 967.0s   | 406.9s       | 1373.9s    |
    | 15 | mediator | 4         | 111_th0.982    | 95.29, 90.97, 93.08  | 967.0s   | 584.7s       | 1551.7s    |

    note: the `cluster time` of CDP increases to less than 7 times when the amount of data increases to 7 times. Nearly all the baseline methods fail in this case with 1.4M data, either out of memory or with low performance.

### Bibtex
```
@inproceedings{zhan2018consensus,
  title={Consensus-Driven Propagation in Massive Unlabeled Data for Face Recognition},
  author={Zhan, Xiaohang and Liu, Ziwei and Yan, Junjie and Lin, Dahua and Change Loy, Chen},
  booktitle={Proceedings of the European Conference on Computer Vision (ECCV)},
  pages={568--583},
  year={2018}
}
```
