## 天池新品实验室: 淘宝直播商品识别
[READ.md in English](ENGLISH_README.md)   
直播带货是淘宝连接商品和消费者的重要方式，买家在观看直播的过程对喜爱的商品进行购买。在单场直播中，主播常常会对成百上千的商品进行展示、试用和介绍，买家如果想购买正在讲解的商品，则需要在该直播关联的商品列表（包含成百上千个商品）中手动去挑选，非常影响用户的购买效率和用户体验。该项目能够通过商品识别的算法，根据直播视频的画面和主播的讲解，自动识别出当前讲解的商品，把对应的购买链接推荐给用户，将大大提升用户的购买体验。本赛题要求选手通过计算机视觉、自然语言处理等人工智能算法，把视频中正在讲解的商品识别出来，提升用户在淘宝直播中的购买体验。  

项目细节请见：[比赛官网](https://tianchi.aliyun.com/competition/entrance/231772/introduction) [微信文章](https://mp.weixin.qq.com/s?__biz=MzUyNzA1OTcxNg==&mid=2247483692&idx=1&sn=34c1737ab81e8d75246ef8dde5549732&chksm=fa041f47cd7396516b2efbceafb6baf26a272847667671b1510106537617d01a2412d71e197c&token=266731819&lang=zh_CN#rd) 
### 1. 环境配置  
请自行安装[mmdetection](https://github.com/open-mmlab/mmdetection)环境配置。  
### 2. 数据准备
模型的训练数据准备主要分为以下两个步骤：  
- ``data_prepare.py``: 获取``images``和``video_images``图片文件夹（分别存放图片库图片和视频库切片图）。同时生成``trainval.json``。  
- ``add_keys.py``: 给标注文件key:``annotaions``下加入标注的``id``。  
- ``create_val.py``: 分层抽样式划分数据集得到``train.json``和``val.json``。  
- ``prepare_img_meta.py``：返回两个文件 - ``./img_meta/<fn>.pth``和``/pos_pair_dict.json``），他们分别是保存各图片的metadata，和各图片的匹配正对图片的文件名们。 
```
python data_prepare.py
python add_keys.py
python create_val.py
python prepare_img_meta.py
```
最后将得到的数据整理成如下文件结构：  
```
├──| mmdetection
├────| data
├───────| coco
├──────────| annotations （coco-like格式）
├──────────────| trainval.json
├──────────────| train.json
├──────────────| val.json
|───────| images (将’images‘和‘video_images’图片文件夹内图片全部复制进来)
```
以上标注文件格式样例如下：
```
{
  "images":
	[
	 {"file_name":"i_'img_name'_'item_id'.jpg", "id":1, "height":1000, "width":1000},
	 {"file_name":"v_'video_id'_'frame_index'.jpg", "id":2, "height":1000, "width":1000},
	 ...
	]
  "annotations":
	[
	 {'image_id': 1,
	 'bbox': [148.0, 74.0, 505.0, 661.0],
	 'category_id': 7,
	 'instance_id': 20004101,
	 'iscrowd': 0,
	 'segmentation': [],
	 'area': 333805,
	 'id': 0}，
	 
	 {'image_id': 2,
	 'bbox': [139.0, 68.0, 512.0, 584.0],
	 'category_id': 7,
	 'instance_id': 20004101,
	 'iscrowd': 0,
	 'segmentation': [],
	 'area': 299008,
	 'id': 1}
	 ...
	]
  "categories":
	[
	 {"id":1, "name":"长袖"}
	 ...
	]
}
```
``./img_meta/<fn>.pth``保存了每个图片的meta data，它是由mmdetection的data_loader所得到的自定义meta data,例如如下：  
```
	# 'img_meta' is a [dict]。
	[{'filename': 'data/coco/images/v_033370_280.jpg',
	 'ori_shape': (960, 540, 3),
	 'img_shape': (1333, 750, 3),
	 'pad_shape': (1344, 768, 3),
	 'scale_factor': 1.3885416666666666,
	 'flip': False,
	 'img_norm_cfg': {'mean': array([123.675, 116.28 , 103.53 ], dtype=float32), 'std': array([58.395, 57.12 , 57.375], dtype=float32),
	 'to_rgb': True}]
```
``/pos_pair_dict.json``记录了数据集中每张图所能匹配的图片名称：
```
# 字典keys是所有图片（包含instance_id=0），但是[正匹配图片们]中只有instance_id>0的正匹配图片。
    {'i_123_1321.jpg':[i_124_131.jpg, i_213_213.jpg, v_213_1231,jpg ...],
     'i_234_1241.jpg':[],
        ...}
```
23类类别信息如下：
```
# 23类类别信息（其中'古风'与'古装'同为第2类）， 最后"categories"内将‘古装’和‘古风’合为‘古风’
CLASS_DICT = collections.OrderedDict({
'短外套':1,
'古风':2, '古装':2,
'短裤':3,
'短袖上衣':4, 
'长半身裙':5, 
'背带裤':6, 
'长袖上衣':7, 
'长袖连衣裙':8, 
'短马甲':9, 
'短裙':10, 
'背心上衣':11, 
'短袖连衣裙':12, 
'长袖衬衫':13, 
'中等半身裙':14, 
'无袖上衣':15, 
'长外套':16, 
'无袖连衣裙':17, 
'连体衣':18, 
'长马甲':19, 
'长裤':20, 
'吊带上衣':21, 
'中裤':22, 
'短袖衬衫':23})
```
### 3. 目标检测模型训练  
模型训练的前期准备工作如下：  
- 从[Model Zoo](https://github.com/open-mmlab/mmdetection/blob/master/docs/MODEL_ZOO.md)里下载预训练模型文件，这里使用Match R-CNN一致的R-50-FPN-1x模型文件，之后将其放入``mmdetection/mmdet/models/pretrained_models``下，并用进行以下操作：  
- **修改预训练模型权重文件**中涉及类别数量的模块内容：运行``cocopth.py``，形成新的模型文件``faster_rcnn_r50_fpn_1x_cls_24.pth``。
- **修改模型配置文件**：``mmdetection/configs/faster_rcnn_r50_fpn_1x.py``修改后命名为``baseline_config.py``。  
先训练目标检测模型：
```
cd mmdetection
CUDA_VISIBLE_DEVICES=4,5 PORT=29501 tools/dist_train.sh configs/baseline_config.py 2 --validate --seed 123456
```

### 4. 匹配模型训练
将``baseline_config.py``内``load_from``下的路径改为目标检测训练好的模型权重的路径。
```
python tools/train_mm.py
```
### 修改点：
1. ``match_rcnn/mmdetection/data/coco``：放置了数据准备相关的代码。  
2. ``match_rcnn/mmdetection/mmdet/core/evaluation/class_name.py``：修改了类别信息。  
3. ``match_rcnn/mmdetection/mmdet/datasets/coco.py``：修改了类别信息。  
4. ``match_rcnn/mmdetection/configs/baseline_config.py``: 修改了``faster_rcnn_r50_fpn_1x.py``配置文件。  
5. ``match_rcnn/mmdetection/tools``：加入了``prepare_img_meta.py``和``train_mm.py``代码以进行匹配网络的数据准备和训练。  
