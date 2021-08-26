---
title:  "SSD: Single Shot MultiBox Detector 리뷰"
excerpt: "YOLO 리뷰 후 SSD도 리뷰해봤다."

categories:
  - Review
tags:
  - SSD
---

# SSD: Single Shot MultiBox Detector            
​                
## Abstract           

SSD는 Single deep neural network 다          

__Multiple feature maps with different resolution__ 을 이용해 다양한 사이즈의 objects들에 대해도 잘 작동한다       

​       

300 x 300 Input으로 74.3% mAP on VOC2007           

​           

[Code](https://github.com/weiliu89/caffe/tree/ssd)         

​              

## 1 Introduction           

기존의 방법들은 too computationally intensive           

> fundamental improvement in speed comes from eliminating bounding box proposals and the subsequent pixel or feature resampling stage

CNN layers로 object class, b.b 위치의 offset 예측
다양한 가로 세로 비율의 predictors 사용하고 이를 다양한 크기의 feature maps 에 적용해 다양한 scale의 detection 가능하게 함

Low resolution에서도 상대적으로 좋은 결과를 얻을 수 있었다

> __The core of SSD is predicting category scores and box offsets for a fixed set of default b.b using small convolutional filters applied to feature maps__

SSD는 simple end-to-end training이 가능하면서 fast and accurate 
<br/>

## 2 The Single Shot Detector (SSD)
SSD는 훈련 시 input image 와 ground truth boxes만 필요로 함  

__Default box (Anchor box)__ 마다 __shape offsets__ 과 __confidences for all object categories((c1, c2, ,,, cp))__ 를 예측함

__Loss__ 는 __localization loss__ 와 __confidence loss__ 의 가중합이다
<br/>

### 2.1 Model 
모델의 초반 layers는 image classification에 구조를 가지고 옴 (이를 base network라 부름)
뒤에 임의의 구조를 추가해 detections을 수행함 

__Multi-scale feature maps for detection__
VGG-16 구조에서 자른 앞의 base network 뒤에 여러 크기의 layers은 다양한 scale detection을 도와준다

__Convolutional predictors for detection__ 
3 x 3 x p  size의 kernel 로 feature cell 들로부터 output을 만든다

__Default boxes and aspect ratios__
각 feature map cell 마다 __default bounding boxes__ 마다의 offset 과 per-class scores 예측한다

c : class scores
4 : Default box로 부터 offset 표현하는 개수 

Faster R-CNN 의 __anchor boxes__ 아이디어를 가져왔지만 다른 크기의 feature map마다 다른 크기의 default boxes 사용함


### 2.2 Training
__Matching Strategy__
어떠한 default boxes가 ground truth 와 맞는지 __jaccard overlap__ 을 통해 비교한다
Jaccard overlap 이 threshold (0.5)를 넘는 박스들을 positive 로 한다

__Training objective__
Loss function : Localization loss 와 confidence loss 의 가중합

Default 로는 둘의 가중치를 동등하게 설정한다  
- YOLO 에서는 confidence loss 를 더 낮게 설정했음 

Faster R-CNN 과 비슷하게 default b.b 로부터의 offset 을 Smooth L1 loss 을 취한다

__Choosing scales and aspect ratios for default boxes__
다른 크기의 feature map은 다른 receptive field size를 가진다
- (논문 Fig 1) 8 x 8 feature map에서는 개를 찾지 못했지만 4 x 4 에서는 찾을 수 있었다

feature map의 크기에 따라 default box 의 크기를 다르게 설정했다
기본적으로 한 feature map cell 마다 6개의 default box 사용 (일부는 4개 사용)
1, 2, 3, 1/2, 1/3 의 가로 세로 비율을 사용했다

__Hard negative mining__
대부분의 default boxes 는 negative 다 
positive samples 와 불균형을 초래한다

confidence loss 를 활용해 상위 박스들을 사용해 3 : 1 의 분포를 만들었다

__Data augmentation__
이미지 전체를 사용하거나 sample patch 를 사용하고 0.5 의 확률로 좌우 뒤집는 작업을 실행했다
<br/>

## 3 Experimental Results
__Base entwork__ 
VGG16을 기반으로 하는데 fc layers 를 conv layers 로 바꿨다

SGD를 사용해 fine-tuning을 진행했다
- Initial learning rate 10^-3
- 0.9 momentum
- 0.0005 weight decay
batch size 32

### 3.1 PASCAL VOC2007
conv4_3, conv7, conv8_2, conv9_2, conv10_2 그리고 conv11_2 을 사용해 location 과 confidence 를 사용했다 (논문의 Figure 2 참고)
이 중 conv10_2, conv11_2는 4개의 default boxes 사용

VOC2007으로 훈련 시켰을 때 Fast R-CNN 보다 더 정확했다

R-CNN과 비교했을 때 localization error가 더 낮았다. 
- SSD는 two disjoint steps이 아니라 object의 모양과 class 를 바로 regress해서

SSD는 한편 similar class 에서 성능이 좋지 않았는데 이는 한 feature map cell 마다 multiple class를 보기 때문이라고 한다

SSD는 small objects 에 대해서 성능이 좋지 않은데 그럴 수 밖에 없는게 very top layers에서는 small objects에 관한 어느 정보도 얻을 수 없기 때문이다

### 3.2 Model analysis 
__Data augmentation is crucial__
Augmentation startegy 로 8.8% mAP 성능 향상

Fast and Faster R-CNN 은 feature pooling step을 사용하기 때문에 augmentation 로 부터 성능 향상이 덜 할 것이다

__More default box shapes is better__
Default ratios을 제거할 수록 성능이 좋지 못했다

__Atrous is faster__
2 x 2 - s2 를 3 x 3 -s1 으로 바꾸면서 빈 공간들을 a trous 알고리즘으로 채웠는데 이는 같은 정확도면서 속도의 향상으로 이어졌다

__Multiple output layers at different resolutions is better__
다양한 크기의 layers들을 제거하면서 비교 실험을 하는데 총 default boxes 수 (8732) 를 일정하게 유지하게 끔 남은 layers에서 boxes 수를 증가시켰다

1 x 1 이나 3 x 3 크기의 feature maps 에서는 large objects를 찾을 만한 large boxes 가 없기 때문에 성능이 좋지 못했다
<br/>

### 3.3 PASCAL VOC2012
VOC2007을 사용해 나온 결과와 동일한 결과를 볼 수 있었다

### 3.4 COCO
COCO 의 objects 들은 대개 PASCAL 의 objects 보다 작아 모든 layers의 default box 크기를 줄였다
- 원래 0.2 부터 0.9 scale 이었으나 0.15 부터 0.9 scale 로 수정

VOC dataset에서 보는 결과와 비슷했다

Faster R-CNN 은 region proposal 와 Fast R-CNN 에서 총 두 번의 box refinement steps이 있기 때문에 small objects에서 SSD 보다 성능이 좋지 않을까 예상했다

### 3.5 Preliminary ILSVRC results
val2 set에 대해 43.4 mAP 성능을 얻었다

### 3.6 Data Augmentation for Small Object Accuracy
PASCAL VOC 와 같은 small data set 에서 data augmentation은 큰 성능 향상 특히 small objects에 대해서도 성능 향상을 이끌었다

### 3.7 Inference time
Confidence threshold (0.01)로 대부분의 boxes을 제거할 수 있었다
이후 jaccard overlap (0.45)로 non-maximum suppresion (nms) 을 진행 해 중복 detection을 제거했다 

80%의 실행 시간은 base network에서 이루어졌다
- VGG16 보다 빠른 네트워크를 사용하면 더욱 빠르게 실행할 수 있다
<br/>

## 4 Related Work 
기존에는 sliding windows 와 region proposal 크게 2 방식이 있었다

이후 R-CNN 에서 selective search region proposal 후 cnn 으로 진행하는 방식을 사용했다

이후 수많은 image crop을 분류하는 것을 해결하고자 SPPnet 에서 spatial pyramid pooling 을 사용했다

이후 Fast R-CNN 에서 confidence 와 b.b regression의 loss를 최소화하고자 했다


또 다른 방식으로 Fast R-CNN이 RPN 과 Fast R-CNN 을 연결시켰다.

SSD는 RPN의 anchor box와 아이디어는 비슷하지만 이를 2개의 step에 process하지 않고 class와 loc을 동시에 예측하게끔 했다


SSD는 또 다른 방식으로 향상시켰는데 바로 proposal step 을 모두 없애고 b.b와 score을 동시에 예측하게끔 하는 방식이다
<br/>

## 5 Conclusions
> A key feature of our model is the use of multi-scale convolutional bounding box outputs attached to multiple feature maps at the top of the network








