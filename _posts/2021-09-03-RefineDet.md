---
title:  "2021-09-03-RefineDet 논문 리뷰"
excerpt: "아이패드에 SSD를 다시 한번 정리해보고 RefineDet 논문을 읽기 시작했다."

categories:
  - TIL
tags:
  - RefineDet
---

# Single-Shot Refinement Neural Network for Object Detection

## Abstract
Two-stage 방식은 정확도에 장점이 있고, one-stage 방식은 속도에 장점이 있다.
__RefineDet__ 은 Two-stage 방법보다 더 accurate라고 one-stage 에 버금가는 efficiency를 갖는다.

RefineDet 은 __two inter-connected modules__ 로 구성되어있다.
__Anchor refinement module__ 과 __Object detection module__

__Anchor refinement module__
- Negative anchors을 줄여준다
- 뒤의 regressor가 잘 작동하게끔 Anchors의 위치와 크기를 수정해 better initialization 을 제공해준다.

__Object detection module__
- 이전 module로부터 refined anchor을 받아 regression과 class을 예측한다.

__Transfer connection block__ 으로 Anchor refinement module 로부터 얻은 features로 object detection module으로 전달해 objects의 위치, 사이즈, 클래스를 예측할 수 있다.

__Multi-task loss function__ 으로 네트워크를 end-to-end로 훈련을 가능케 했다.

PASCAL VOC 2007, 2012, MS COCO 에서 SOTA
<br/>
-----------

## 1. Introduction
Two-stage approach 보다 one-stage approach 가 정확도가 잘 안 나오는 이유 중 하나는 __class imbalance problem__ .

Two-stage 가 갖는 3 가지 장점
- Using two-stage structure with sampling heuristics to handle class imbalance.
- Using two-step cascade to regress the object box parameters.
- Using two-stage features to describe the objects.

__요약__
RefineDet은 two inter-connected modules (__ARM, ODM__ ) 으로 구성된 __one-stage__ framework다.
__TCB__ 로 ARM로부터의 features를 ODM으로 전달해 object의 대한 정보(위치, 크기, 클래스)를 예측하게끔 해준다.
<br/>
--------
## 2. Related Work
__Classical Object Detectors.__
- __Sliding-window__ 기반
- Hand-crafted features와 classifiers를 사용

__Two-Stage Approach.__
첫 번째 stage: Object proposals 생산
두 번째 stage: Object regions, class label 결정

__One-stage Approach.__
YOLO, SSD (DSSD, DSOD)
Class imbalance problem을 해결하고자
- Re-designing the loss function or classification strategies.
<br/>
------------
## 3. Network Architecture
SSD와 비슷하게, CNN을 거쳐 __fixed number of bounding boxes__ 와 __class별 score__ 를 만들고, __non-maximum suppresion__을 통해 최종 결과를 만든다.

__ARM__ 은 two base networks(VGG-16, ResNet-101)에서 classification layers를 제거하고 추가적인 layers를 붙였다.
__ODM__ 은 TCB로부터의 결과를 prediction layers(conv layer(3x3 kernel))에 input으로 사용해 class별 score와 shape offset을 예측한다.

Three core components in RefineDet
- TCB
- Two-step cascaded regression 으로 objects의 위치와 사이즈을 regress한다.
- Negative anchor filtering

__Transfer Connection Block.__
ARM으로부터 나온 features들을 ODM이 요구하는 형식으로 __convert__

(1) Detection accuaracy을 향상시키기 위해 __high-level feature map__ 을 거침
(2) ARM과 ODM의 dimensions을 맞추기 위해 high-level feature map을 거치고 나온 걸 __deconvolution operation__ 실행

(1) 과 (2)을 __element-wise way__로 더함

conv layer를 또다시 거침

__Two-Step Cascaded Regression__
최근의 one-stage methods는 one-step regression을 사용했는데 이는 특히 __small objects__ 에 대해 부정확했다.

먼저, __ARM__ 에서 feature maps의 각 cell 마다 refined anchor boxes와 original tiled _n_ anchor boxes 와의 __four offsets__ 과
박스에 object의 존재를 알려주는 __two confidence scores__ 를 예측한다.

ODM에서는 _c_ class scores 와 four accurate offsets 를 계산
각 refined anchor boxes 마다 __c + 4__ outputs

SSD는 deafult boxes를 directly 사용하지만,
RefineDet은
- ARM에서 refined anchor boxes 만든다
- ODM에서 refined anchor boxes들을 input으로 받아 class score와 offset 계산

__Negative Anchor Filtering__
Train 과정 중에는 refined anchor box의 negative confience가 preset threshold보다 높으면 ODM으로 넘기지 않는다.
즉, refined positive anchor boxes들만 ODM에서 훈련하게 된다.

Inference 과정 중에는 refined hard negative anchor boxes는 ODM에서 사용되지 않는다.
<br/>
-------------
## 4. Training and Inference
__Data Augmentation.__
SSD와 같은 방법으로
- 랜덤으로 확장한 상태에서 crop
- 랜덤으로 photometric distortion
- 랜덤으로 뒤집기

__Backbone Network.__
VGG-16과 ResNet-101 을 backbone networks로 사용했다.

SSD처럼,
VGG-16의 fc6과 fc7을 conv_fc6과 conv_fc7로 바꿨다.
또한,
conv4_3과 conv5_3은 다른 layer와 다른 feature scales을 가지기 때문에 L2 normalization을 사용했다.

잘린 VGG-16의 뒤에 two extra conv layers를 추가하고 (conv6_1, conv6_2),
잘린 ResNet-101에 one extra residual block을 추가했다. (res6)

__Anchors Design and Matching.__
VGG-16 base network에서는
conv4_3, conv5_3, conv_fc7 와 conv6_2 layers로부터 objects의 위치, 크기, confidence를 예측한다.
ResNet-101 base network에서는
res3b3, res4b22, res5c, 와 res6가 예측에 사용된다.


## 이해가 잘 되지 않는 부분
total stride sizes: 8, 16, 32, and 64 pixels <- 어디에 쓰이는지 잘 모르겠네,,,
> Each feature layer is associated with one specific scale of anchors and three aspect ratios. We follow the design of anchor scales over different scales of anchors have the same tiling density on the image.

각 g.t와 가장 높은 jaccard overlap score를 갖는 anchor box를 match한다.
각 anchor boxes를 g.t 중 overlap이 0.5보다 높기만 하면 match한다.

__Hard Negative Mining.__
SSD 처럼
negative anchor boxes 중 top loss values를 갖는 일부 boxes를 이용해 negatives 와 positives의 비율을 3:1로 맞춰 class imbalance를 해결하려 했다.

__Loss Function.__