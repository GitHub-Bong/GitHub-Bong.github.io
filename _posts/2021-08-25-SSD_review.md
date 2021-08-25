---
title:  "2021-08-25 SSD 논문 리뷰"
excerpt: "SSD 논문 리뷰를 시작했다."

categories:
  - TIL
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





