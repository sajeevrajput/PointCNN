# PointCNN

Created by <a href="http://yangyan.li" target="_blank">Yangyan Li</a>,<a href="http://rbruibu.cn" target="_blank"> Rui Bu</a>, <a href="http://www.mcsun.cn" target="_blank">Mingchao Sun</a>, and <a href="http://www.cs.sdu.edu.cn/~baoquan/" target="_blank">Baoquan Chen</a> from Shandong University.

## Introduction

PointCNN is a simple and general framework for feature learning from point cloud, which refreshed five benchmark records in point cloud processing, including:

* classification accuracy on ModelNet40 (91.7%)
* classification accuracy on ScanNet (77.9%)
* segmentation part averaged IoU on ShapeNet Parts (86.13%)
* segmentation mean IoU on S3DIS (62.74%)
* per voxel labelling accuracy on ScanNet (85.1%).

See our <a href="http://arxiv.org/abs/1801.07791" target="_blank">research paper on arXiv</a> for more details.

## Code Organization
The core X-Conv and PointCNN architecture are defined in [pointcnn.py](pointcnn.py).

The network/training/data augmentation hyper parameters for classification tasks are defined in [pointcnn_cls](pointcnn_cls), for segmentation tasks are defined in [pointcnn_seg](pointcnn_seg).

### Explanation of X-Conv and X-DeConv Parameters
Take the xconv_params and xdconv_params from [shapenet_x8_2048_fps.py](pointcnn_seg/shapenet_x8_2048_fps.py) for example:
```
# K, D, P, C
xconv_params = [(8, 1, -1, 32 * x),
                (12, 2, 768, 32 * x),
                (16, 2, 384, 64 * x),
                (16, 6, 128, 128 * x)]

# K, D, pts_layer_idx, qrs_layer_idx
xdconv_params = [(16, 6, 3, 2),
                 (12, 6, 2, 1),
                 (8, 6, 1, 0),
                 (8, 4, 0, 0)]
```
Each element in xconv_params is a tuple of (K, D, P, C), where K is the neighborhood size, D is the dilation rate, P is the representative point number in the output (-1 means all input points are output representative points), and C is the output channel number. Each element specifies the parameters of one X-Conv layer, and they are stacked to create a deep network.

Each element in xdconv_params is a tuple of (K, D, pts_layer_idx, qrs_layer_idx), where K and D have the same meaning as that in xconv_params, pts_layer_idx specifies the output of which X-Conv layer (from the xconv_params) will be the input of this X-DeConv layer, and qrs_layer_idx specifies the output of which X-Conv layer (from the xconv_params) will be forwarded and fused with the output of this X-DeConv layer. The P and C parameters of this X-DeConv layer is also determined by qrs_layer_idx. Similarly, each element specifies the parameters of one X-DeConv layer, and they are stacked to create a deep network.


## PointCNN Usage

Here we list the commands for training/evaluating PointCNN on classification and segmentation tasks on multiple datasets.

* ### Classification

  * #### ModelNet40
  ```
  cd data_conversions
  python3 ./download_datasets.py -d modelnet
  cd ../pointcnn_cls
  ./train_val_modelnet.sh -g 0 -x modelnet_x2_l4
  ```
  
  * #### ScanNet
  Please refer to <http://www.scan-net.org/>  for downloading ScanNet task data and scannet_labelmap, and refer to https://github.com/ScanNet/ScanNet/tree/master/Tasks/Benchmark for downloading ScanNet benchmark files:
  
  scannet_dataset_download
  
  |_ data
  
  |_ scannet_labelmap
  
  |_ benchmark

  ```
  cd ../data/scannet/scannet_dataset_download/
  mv ./scannet_labelmap/scannet-labels.combined.tsv ../benchmark/

  #./pointcnn_root
  cd ../../../pointcnn/data_conversions
  python scannet_extract_obj.py -f ../../data/scannet/scannet_dataset_download/data/ -b ../../data/scannet/scannet_dataset_download/benchmark/ -o ../../data/scannet/cls/
  python prepare_scannet_cls_data.py -f ../../data/scannet/cls/
  cd ../pointcnn_cls/
  ./train_val_scannet.sh -g 0 -x scannet_x2_l4.py
  ```

  * #### tu_berlin
  ```
  cd data_conversions
  python3 ./download_datasets.py -d tu_berlin
  cd ../pointcnn_cls
  ./train_val_tu_berlin.sh -g 0 -x tu_berlin_x2_l5
  ```
  
  * #### quick_darw
  Note that the training/evaluation of quick_draw requires LARGE RAM, as we load all stokes into RAM and converting them into point cloud on-the-fly.
  ```
  cd data_conversions
  python3 ./download_datasets.py -d quick_draw
  cd ../pointcnn_cls
  ./train_val_quick_draw.sh -g 0 -x quick_draw_full_x2_l6
  ```
  
  * #### MNIST
  ```
  cd data_conversions
  python3 ./download_datasets.py -d mnist
  cd ../pointcnn_cls
  ./train_val_mnist.sh -g 0 -x mnist_x2_l5
 	```

  * #### CIFAR-10
  ```
  cd data_conversions
  python3 ./download_datasets.py -d cifar10
  cd ../pointcnn_cls
  ./train_val_cifar10.sh -g 0 -x cifar10_x2_l4
  ```
  
* ### Segmentation

	We use farthest point sampling (the implementation from <a href="https://github.com/charlesq34/pointnet2" target="_blank">PointNet++</a> in segmentation tasks. Compile FPS before the training/evaluation:
	```
	cd sampling
	bash tf_sampling_compile.sh
	```

  * #### ShapeNet
  ```
  cd data_conversions
  python3 ./download_datasets.py -d shapenet_partseg
  cd ../pointcnn_seg
  ./train_val_shapenet.sh -g 0 -x shapenet_x8_2048_fps
  ./test_shapenet.sh -g 0 -x shapenet_x8_2048_fps -l ../../models/seg/pointcnn_seg_shapenet_x8_2048_fps_xxxx/ckpts/iter-xxxxx -r 10
  cd ../evaluation
  python3 eval_shapenet_seg.py -g ../../data/shapenet_partseg/test_label -p ../../data/shapenet_partseg/test_data_pred_10
  ```

  * #### S3DIS
	Please refer to [data_conversions](data_conversions/README.md) for downloading S3DIS, then:
  ```
  cd data_conversions/split_data
  python3 s3dis_prepare_label.py
  python3 s3dis_split.py
  cd ..
  python3 prepare_multiChannel_seg_data.py -f ../../data/S3DIS/out_part_rgb/ -c 6
  ./train_val_s3dis.sh -g 0 -x s3dis_x8_2048_fps_k16
  ./test_shapenet.sh -g 0 -x s3dis_x8_2048_fps_k16 -l ../../models/seg/s3dis_x8_2048_fps_k16_xxxx/ckpts/iter-xxxxx -r 4
  cd ../evaluation
  python3 eval_s3dis.py
  ```

  * #### ScanNet
	Please refer to [data_conversions](data_conversions/README.md) for downloading ScanNet, then:
  ```
  cd data_conversions/split_data
  python3 scannet_split.py
  cd ..
  python3 prepare_multiChannel_seg_data.py -f ../../data/scannet/scannet_split_dataset/
  cd ../pointcnn_seg
  ./train_val_scannet.sh -g 0 -x scannet_x8_2048_k8_fps
  ./test_scannet.sh -g 0 -x scannet_x8_2048_k8_fps -l ../../models/seg/pointcnn_seg_scannet_x8_2048_k8_fps_xxxx/ckpts/iter-xxxxx -r 4
  cd ../evaluation
  python3 eval_scannet.py
  ```
