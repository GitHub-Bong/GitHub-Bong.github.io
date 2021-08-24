---
title:  "2021-08-24 YOLO 논문 리뷰"
excerpt: "어제부터 시작한 YOLO 논문 리뷰를 마쳤다."

categories:
  - Blog
tags:
  - Blog
---

# You Only Look Once: Unified, Real-Time Object Detection               

​                    

## Abstract             

Object Detection problem 을 regression problem 처럼 생각하고 접근          

> __Single neural network__ predicts __bounding boxes__ and __class probabilities__ directly from __full images__ in __one evaluation__ .          

End-to-end   pipeline              

YOLO is __Fast!__           

- base YOLO model은 45 fps, Fast YOLO는 155 fps 가능        

Makes more localization errors!    But less likely to predict false positives on background.            

- __아마 뒤에 나오겠지만 localization error가 많은 이유는 image를 S x S grid cells로 나눠 각 cell 마다 one class 만 예측하기 때문이지 않을까?__            
- __한 번에 이미지 전체를 보기 때문에 일부만 보는 다른 모델들보다 background error가 적지 않을까?__            

​          

<br/>

## 1. Introduction           

Deformable parts models (DPM)  use __sliding window approach__              

R-CNN use __region proposal__            

- 처음에 generate potential b.b (bounding boxes)         
- Each boxes 마다 어떤 class 인지 예측           
- refine b.b           
- __Slow (2-staged)  & must be trained seperately__           

​             

YOLO 는 한 번에 __b.b 좌표__ 와 __class probabilities__  예측            

> A single convolutional network __simultaneously__ predicts __multiple bounding boxes and class probabilities for those boxes__           

​             

[real time 에서 실행한 영상](http://pjreddie.com/yolo/)            

​                          

> YOLO reasons globally about the image                  

See __entire image__   (Unlike sliding window and region proposal)            

- __그래서 Fast R-CNN 은 background match error 가 많은데 (can't see larger context), YOLO는 전체 이미지를 한 번에 보니 background error가 적을 수 밖에__          

​                 

> YOLO learns generalizable representations of objects            

새로운 domain 에 가져가도 다른 모델들 보다 성능 저하가 적다                   

​              

YOLO는 빠르지만 __작은 objects를 잘 찾지 못한다__            

- grid cell 마다 하나의 class 를 예측해서 한 cell 에 작은 두 objects가 있으면 잘 감지하지 못 할듯                  

​                   

<br/>

## 2. Unified Detection              

> predicts all bounding boxes across all classes for an image simultaneously             

​                   

각 b.b을 예측할 때 전체 의미에서 나온 features를 사용한다                            

​                      

Image를 __S x S grid로 나눈다__             

각 grid cell 마다 __B bounding boxes 와 각 b.b의 confidence scores__ 를 예측한다                

- cell 에 object가 없으면 confidence scores = 0         
- confidence : __(박스에 object가 들어있을 확률) x IOU__            
- 각 bounding box 는 5 predictions : x, y, w, h, confidence                  
  - (x, y)는 박스의 중앙 의미 w, h는 박스의 가로, 세로              

각 grid cell 마다 __C   conditional class probabilities, Pr(Class_i|Objects)__ 도  예측한다             

- 각 cell 마다 1 class만 예측

​                     

Test time 때에는 __C__ 와 __박스마다 confidence__  곱해서 __Pr(Class_i) * IOU__                 

- __Pr(Class_i) * IOU__ : __박스에 예측한 class가 있을 확률과 object에 box가 얼마나 잘 fit한지__                   

​                      

final prediction : __S x S ( B * 5 + C)__ tensor             

- 한 셀 마다 b.b 예측하고 각 b.b 가 5 predictions        
- class 별 C 도 예측       
- 논문에서는 S = 7, B = 2, C = 20 사용  7 x 7 x (30) tensor                  

​             

<br/>

### 2.1 Network Design             

Pascal VOC detection dataset에 evaluate         

​         

GoogLeNet for classification 의 구조 가지고 왔다          

- 24 conv layers, 2 fc layers          
- inception modules 대신 1 x 1 layer -> 3 x 3 layer         
- Fast YOLO는 9 conv layers 사용        
- Conv layers pretrain  할 때는 GoogleNet input 그대로 224 x 224 input size          
- Detection 할 때는 448 x 448 input size  이유는 뒤에           

​             

<br/>

## 2.2 Training             

ImageNet 1000-class dataset에 앞에 __20 conv layers를 pretrain__ 했다            

​          

Classification 보다 detection 은 fine-grained visual info 를 필요로 해서 224 x 224 로 __448 x 448__ 사이즈로 증가시켰다            

​                  

Pretrained  된 20 conv layers 뒤에 random initialized 된 __4 conv layers 와 2 fc layers__  추가 했다              

​               

Final layer predicts both __class probabilities__ and __b.b coordinates__          

- b.b 의 가로 세로를 image의 가로 세로에 normalize 해 0 ~ 1 사이 값으로 표현        
- b.b 의 x,y  좌표도 각 grid cell에서의 offset으로 parametrize (마찬가지로 0 ~ 1 사이 값으로 표현)             



final layer 는 linear activation function 사용          

나머지 layers 는 leaky ReLu 사용           

​             

Sum-squared error 사용 (easy to optimize)          

Localization error 와 classification error 동일하게 하는데 이는 not ideal           

- 대부분의 grid cell 에서는 object 가 없기 때문에 이들의 confidence는 0 이는 다른 cell의 gradients 에 큰 영향을 줘 모델 unstable        

__Increase__ loss from b.b coordinates predictions and __decrease__ loss from confidence predictions             

​             

__큰 박스의 작은 크기 변화는 작은 박스의 작은 크기 변화 보다 가중치를 적게 하기 위해__ 가로 세로의 제곱근을 예측         

​                   

Train 중 Each b.b predictor 마다 실제 박스와 가장 높은 IOU를 가지는 하나의 object를 예측하게끔         

​         

Loss function:       

- 해당 cell에 object가 있어야만 penalizes classification error       
- 가장 IOU가 높은 predictor만 penalizes b.b coordinates error            

​             

Epochs : 135       

Batch size : 64        

Momentum : 0.9          

Decay : 0.0005          

Learning rate : 10 ^-3  -> 10^-2 -> 10^-3 -> 10^-4           

​              

Overfitting 피하기 위해           

- 첫 fc layer 뒤에 Dropout 0.5       
- Data augmentation          

​          

<br/>

### 2.3 Inference             

Multiple grid cells 에 퍼져있는 objects는 중복 감지 됨               

-> 이를 해결하기 위해 __Non-maximal suppression__                   

​                  

<br/>

### 2.4 Limitations of YOLO            

Strong spatial constraints        

- 새 떼 같은 작은 objects들로 이루어진 group 잘 감지 못 함     
- 훈련된 data에 없는 새로운 비율이나 구성의 object에 약함        
- __작은 bb에서의 에러와 큰 bb에서의 에러를 동등하게 여긴다__           

​               

<br/>

## 3. Comparison to Other Detection Systems             

이전 model들      

Start by extracting features from input images        

Classifiers or localizers, identify objects in feature space (by sliding window or regions in image)         

​              

__Deformable parts models.__            

Sliding window 방식 + Disjoint pipeline        

​             

__R-CNN__        

Region proposals       

Selective search로 b.b 만들고, cnn 거쳐서 SVM으로 box score, 등등 복잡해서 한 image당 40s             

__YOLO도 각 grid cell마다 b.b 만들고 confidence score 만들지만 grid 방식으로 중복 감지가 더 적고 b.b 갯수가 더 적다 ( 2000 vs 98 )__              

​               

__Other Fast Detectors__         

Fast and Faster R-CNN : neural network로 region 생성했지만 real time으로 하기에는 아직 느림     

DPM 방식도 느림            

​          

__Deep MultiBox__         

Single object detection은 수행 가능  그러나 general 한 건 불가       

Not a complete -> Need further image path classification          

​            

__OverFeat__             

Disjoint Pipeline + Local info만 사용해 예측해 global context 불가     

​           

__MultiGrasp__            

Grasp detection 의 아이디어와 비슷하나 YOLO 가 훨씬 복잡한 task 수행     

Grasp 은 오직 grasping에 적합한 region만 찾음             

​                

<br/>

## 4. Experiments            

PASCAL VOC 2007로 다른 real-time detection 과 비교해봄         

Fast R-CNN에 비해 __background false positives에 강하고, new domain에서도 성능 좋음__           

​        

### 4.1 Comparison to Other Real-Time Systems                

YOLO using VGG-16 은 더 정확하지만 느림      

Fastest DPM, R-CNN minus R, Fast R-CNN 다 real-time에 무리       

최근 Faster R-CNN은 selective search를 neural network로 바꿨지만 느리거나 꽤 빠르더라도 부정확했음            

​            

<br/>

### 4.2 VOC 2007 Error Analysis        

YOLO와 Fast R-CNN을 비교     

Dataset : VOC 2007         

​          

5 Types or error:             

- Correct         
- Localization       
- Similar          
- Other            
- Background           

​          

__YOLO는 object 위치 잘 찾지 못함 그러나 Background false positive (없는데 object가 있다고 하는) error는 적음__             

​          

<br/>

### 4.3 Combining Fast R-CNN and YOLO         

Fast R-CNN에 YOLO 결합했더니 성능 좋아짐        

(R-CNN에서 예측한 b.b 를 YOLO에서도 비슷한 b.b로 예측하게 되면 boost 줌)            

​          

<br/>

### 4.4 VOC 2012 Results            

YOLO 모델은 VOC 2012에서 성능이 57.9% mAP         

Fast R-CNN + YOLO 은 5등           

​             

<br/>

### 4.5 Generalizability: Person Detection in Artwork       

R-CNN 은 VOC 2007에서는 좋은 성능을 보였으나 artwork에서는 성능이 안 좋았음  

- Selective search 가 natural images로 학습되어서           

DPM은 둘다 성능이 비슷비슷했음  (애초에 VOC에서 성능이 그렇게 좋지 못했음)           

​               

YOLO는 성능이 둘다 좋았음          

- Artwork 과 natural image는 pixel level 에서는 매우 다르지만 object level에서는 모양과 크기가 비슷하기 때문에 두 dataset 모두에서 좋은 성능을 보일 수 있음            

​           

<br/>

## 5. Real-Time Detection In the Wild                 

웹켐으로부터 들어오는 이미지들을 YOLO로 processing해 tracking system 처럼 실행해봤다         

[관련 코드와 영상](http://pjreddie.com/yolo)          

​           

## 6. Conclusion          

YOLO는 unified model       

새로운 domain도 곧잘 detect 할 수 있다         

​         





