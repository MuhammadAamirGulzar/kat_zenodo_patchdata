## KAT Baseline — Zenodo Patch Dataset

Applies the Kernel Attention Transformer (KAT) architecture to a pre-patched Zenodo whole-slide-image dataset as one of the feature-aggregation baselines evaluated in [MSI/MSS status prediction from WSI](https://github.com/muhammadaamirgulzar/colonmsi-fm) using foundation model embeddings.

The KAT model, training loop, and kernel-based graph attention mechanism are Yushan Zheng's implementation of the paper cited below. The additions here are the dataset wiring for pre-extracted Zenodo patches (`zenodo_patches_names.csv`, `patches_with_label.csv`, `ground truth.xlsx`) in place of the raw-slide pipeline the original KAT repo expects.

### Data preparation

Generate the dataset configuration file:

```bash
python dataset/configure_dataset.py
```

### Train

Single GPU:

```bash
CONFIG_FILE='configs/tcga_lung.yaml'
WORKERS=8
GPU=0

python cnn_sample.py --cfg $CONFIG_FILE --num-workers $WORKERS
for((FOLD=0;FOLD<5;FOLD++));
do
    python cnn_train_cl.py --cfg $CONFIG_FILE --fold $FOLD \
        --epochs 21 --batch-size 100 --workers $WORKERS \
        --fix-pred-lr --eval-freq 2 --gpu $GPU

    python cnn_wsi_encode.py --cfg $CONFIG_FILE --fold $FOLD \
        --batch-size 512 --num-workers $WORKERS --gpu $GPU

    python kat_train.py --cfg $CONFIG_FILE --fold $FOLD --node-aug \
        --num-epochs 200 --batch-size 32 --num-workers $WORKERS --weighted-sample \
        --eval-freq 5 --gpu $GPU
done
```

Multi-GPU:

```bash
CONFIG_FILE='configs/tcga_lung.yaml'
WORKERS=8
WORLD_SIZE=1

python cnn_sample.py --cfg $CONFIG_FILE --num-workers $WORKERS

for((FOLD=0;FOLD<5;FOLD++));
do
    python cnn_train_cl.py --cfg $CONFIG_FILE --fold $FOLD \
        --epochs 21 --batch-size 400 --workers $WORKERS \
        --fix-pred-lr --eval-freq 2 \
        --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size $WORLD_SIZE --rank 0

    python cnn_wsi_encode.py --cfg $CONFIG_FILE --fold $FOLD \
        --batch-size 512 --num-workers $WORKERS \
        --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size $WORLD_SIZE --rank 0

    python kat_train.py --cfg $CONFIG_FILE --fold $FOLD --node-aug \
        --num-epochs 200 --batch-size 128 --num-workers $WORKERS --weighted-sample --eval-freq 5 \
        --dist-url 'tcp://localhost:10001' --multiprocessing-distributed --world-size $WORLD_SIZE --rank 0
done
```

### Kernel contrastive learning variant (KCL)

Zheng et al.'s extended variant adds a contrastive representation-learning module on the kernels for better accuracy and generalization. Run `katcl_train.py` in place of `kat_train.py` to use it:

```bash
python katcl_train.py --cfg $CONFIG_FILE --fold $FOLD \
        --num-epochs 200 --batch-size 32 --num-workers $WORKERS --weighted-sample \
        --eval-freq 5 --gpu $GPU
```

### Citation

This repository builds on:

```
@inproceedings{zheng2022kernel,
    author    = {Yushan Zheng, Jun Li, Jun Shi, Fengying Xie, Zhiguo Jiang},
    title     = {Kernel Attention Transformer (KAT) for Histopathology Whole Slide Image Classification},
    booktitle = {Medical Image Computing and Computer Assisted Intervention -- MICCAI 2022},
    pages     = {283--292},
    year      = {2022}
}

@article{zheng2023kernel,
    author    = {Yushan Zheng, Jun Li, Jun Shi, Fengying Xie, Jianguo Huai, Ming Cao, Zhiguo Jiang},
    title     = {Kernel Attention Transformer for Histopathology Whole Slide Image Analysis and Assistant Cancer Diagnosis},
    journal   = {IEEE Transactions on Medical Imaging},
    year      = {2023}
}
```

Original implementation: [zhengyushan/kat](https://zhengyushan.github.io/pdf/article_zheng_tmi_2023.pdf). License terms from the upstream project are preserved in [LICENSE](LICENSE).
