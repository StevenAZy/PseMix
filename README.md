# Pseudo-Bag Mixup Augmentation for Multiple Instance Learning-Based Whole Slide Image Classiffcation (IEEE-TMI 2024)

[[arXiv preprint]](https://arxiv.org/abs/2306.16180) | [[IEEE-TMI]](https://ieeexplore.ieee.org/abstract/document/10385148) 

## Overview

<img src="./doc/conceptual-psemix.png" width="40%" align='left' />

*TL;DR*: 
> Multiple instance learning (MIL) has become one of the most important frameworks for gigapixel Whole Slide Images (WSIs). In current practice, most MIL networks often face two unavoidable problems in training: i) insufficient WSI data and ii) the sample memorization inclination inherent in neural networks. These problems may hinder MIL models from adequate and efficient training, suppressing the continuous performance promotion of classification models on WSIs. Inspired by the basic idea of Mixup, this paper proposes a new Pseudo-bag Mixup (PseMix) data augmentation scheme to improve the training of MIL models. This scheme generalizes the Mixup strategy for general images to special WSIs via pseudo-bags so as to be applied in MIL-based WSI classification. Cooperated by pseudo-bags, our PseMix fulfills the critical size alignment and semantic alignment in Mixup strategy. Moreover, it is designed as an efficient and decoupled method, neither involving time-consuming operations nor relying on MIL model predictions. Comparative experiments and ablation studies are specially designed to evaluate the effectiveness and advantages of our PseMix. Experimental results show that PseMix could often assist state-of-the-art MIL networks to refresh their classification performance on WSIs. Besides, it could also boost the generalization performance of MIL models in special test scenarios, and promote their robustness to patch occlusion and label noise. 

## Key features

Applying PseMix (as a data augmentation method) to MIL networks, e.g., ABMIL, DSMIL, and TransMIL, could 
- **improve network performance** with minimal extra computational costs, without introducing any complicated techniques
- help the network to obtain **better generalization and robustness** (against patch occlusion and noisy label learning)

## Minimal code example (pseudo-code)

```python
# generate_pseudo_bags: function for dividing WSI bags into pseudo-bags
# ALPHA: the hyper-parameter of Beta distribution
# N: the number of pseudo-bags in each WSI bag
# PROB_MIXUP: random mixing parameter for determining the proportion of mixed bags. 
for (X, y) in loader: # load a minibatch 
    n_batch = X.shape[0] # with `n_batch` WSI bags (samples)

    # 1. dividing each bag into `N` pseudo-bags
    X = generate_pseudo_bags(X)

    new_idxs = torch.randperm(n_batch)
    # draw a mixing scale from Beta distribution
    lam = numpy.random.beta(ALPHA, ALPHA) 
    lam = min(lam, 1.0 - 1e-5) # avoid numerical overflow when transforming it into discrete ones
    lam_discrete = int(lam * (N + 1)) # transform into discrete values

    # 2. pseudo-bag-level Mixup generates samples (new_X, new_y)
    new_X, new_y = [], []
    for i in range(n_batch):
    	# randomly select pseudo-bags according to `lam_discrete`
        masked_bag_A = select_pseudo_bags(X[i], lam_discrete) # select `lam_discrete` pseudo-bags
        masked_bag_B = select_pseudo_bags(X[new_idxs[i]], N - lam_discrete) # select `n-lam_discrete` pseudo-bags

        # random-mixing mechanism for two purposes: more data diversity and efficient learning on mixed samples.
        if np.random.rand() <= PROB_MIXUP:
            mixed_bag = torch.cat([masked_bag_A, masked_bag_B], dim=0) # instance-axis concat
            new_X.append(mixed_bag)
            mix_ratio = lam_discrete / N
        else:
            masked_bag = masked_bag_A 
            new_X.append(masked_bag)
            mix_ratio = 1.0

        # target-level mixing
        new_y.append(mix_ratio * y[i] + (1 - mix_ratio) * y[new_idxs[i]]) 

    # 3. minibatch training
    minibatch_training(new_X, new_y)
```

Our implementation roughly follows the pseudo-codes above. More details could be found by the following codes:

- [generate_pseudo_bags](https://github.com/liupei101/PseMix/blob/main/utils/core.py#L146C13-L146C13).
- [pseudo-bag-level Mixup](https://github.com/liupei101/PseMix/blob/main/utils/core.py#L13C10-L13C10).
- [training with mixed labels](https://github.com/liupei101/PseMix/blob/main/model/clf_handler.py#L381).
- [weighted loss for mixed samples](https://github.com/liupei101/PseMix/blob/main/model/clf_handler.py#L407), following [the implementation of Mixup](https://github.com/facebookresearch/mixup-cifar10).

## Running the code

Using the following command to load running configurations from a yaml file and train the model:
```bash
python3 main.py --config config/cfg_clf_mix.yml --handler clf --multi_run
```

The configurations that we need to pay attention are as follows:
- dataset related (we process WSIs with [CLAM](https://github.com/mahmoodlab/CLAM))
  - `path_patch`: the directory path to patch files. 
  - `path_table`: the file path of a csv table that contains WSI IDs and their label information.
  - `data_split_path`: the file path of a npz file that stores data splitting information. 
- network related
  - `net_dims`: the setting of embedding dimension, e.g., `1024-256-2`.
  - `backbone`: network backbone, one of `ABMIL`, `DSMIL`, and `TransMIL`.
- pseudo-bag dividing related
  - `pseb_dividing`: the method used to divide instances, one of `proto`, `kmeans`, and `random`.
  - `pseb_n`: the number of pseudo-bags for each WSI bag, 30 by default.
  - `pseb_l`: the number of phenotypes, 8 by default.
  - `pseb_iter_tuning`: the number of fine-tuning iterations, 8 by default.
  - `pseb_mixup_prob`: the probability of random-mixing.
- Mixup related
  - `mixup_type`: the method of Mixup, `psebmix` by default.
  - `mixup_alpha`: the parameter of beta distribution, i.e., the value of alpha. 

Other configurations are explained in `config/cfg_clf_mix.yml`. They could remain as before without any changes. 

## Performance

AUC, averaged over 5 folds, is as follows:

| Network | BRCA | NSCLC | RCC | Average                                  |
|---------|------|-------|-----|------------------------------------------|
| ABMIL   | 87.05 | 92.23 | 97.36 | 92.21 |
| ABMIL **w/ PseMix**   | 89.49 | 93.01 | 98.02 | 93.51 |
| DSMIL   | 87.73 | 92.99 | 97.65 | 92.79 |
| DSMIL **w/ PseMix**   | 89.65 | 93.92 | 97.89 | 93.82 |
| TransMIL   | 88.83 | 92.14 | 97.88 | 92.95 |
| TransMIL **w/ PseMix**   | 90.40 | 93.47 | 97.76 | 93.88 |

## Training curves

Training curves (training and test AUC, exported from wandb) are listed as follows. Solid lines indicate training with PseMix, and dashed ones are those vanilla models without PseMix.   

| Model                                                                  | Wandb training curves                                  |
|------------------------------------------------------------------------|--------------------------------------------------------|
| [ABMIL](https://proceedings.mlr.press/v80/ilse18a.html)                |![](docs/wandb-abmil-train.png)          |
| [DSMIL](https://openaccess.thecvf.com/content/CVPR2021/papers/Li_Dual-Stream_Multiple_Instance_Learning_Network_for_Whole_Slide_Image_Classification_CVPR_2021_paper.pdf)         | ![](docs/wandb-dsmil-train.png)   |
| [TransMIL](https://openreview.net/forum?id=LKUfuWxajHc)     |![](docs/wandb-transmil-train.png)        |

## Citation

If you find this work helps your research, please consider to cite the paper:
```txt
@article{liu10385148,
  author={Liu, Pei and Ji, Luping and Zhang, Xinyu and Ye, Feng},
  journal={IEEE Transactions on Medical Imaging}, 
  title={Pseudo-Bag Mixup Augmentation for Multiple Instance Learning-Based Whole Slide Image Classification}, 
  year={2024},
  volume={},
  number={},
  pages={1-1},
  doi={10.1109/TMI.2024.3351213}
}
```
or `P. Liu, L. Ji, X. Zhang and F. Ye, "Pseudo-Bag Mixup Augmentation for Multiple Instance Learning-Based Whole Slide Image Classification," in IEEE Transactions on Medical Imaging, doi: 10.1109/TMI.2024.3351213.`