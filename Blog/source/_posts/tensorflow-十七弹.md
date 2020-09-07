---
title: tensorflow-十七弹
date: 2017-09-26 20:29:58
tags: tensorflow学习
categories: 学习
mathjax: true
---
终于到了十七弹，虽然理论基础还没有介绍完，但是我们还是赶快进行进一步的学习，这一次我们主要来说一个很有用的工具，Fast-RCNN，为什么要说这个，因为目标识别要用到它，所以我们不得不在接下来的一段时间中都对这个RCNN进行研究，当然，我们已经了解了神经网络的基础知识，了解了卷积神经网络，也了解到了循环神经网络，卷积神经网络更多的是用于目标的识别，而循环神经网络更多的是用于对序列数据进行处理，而我们现在要做的是在影像上对目标进行检测，当然咯，最简单的办法就是拿一个框在影像上移动，然后框出目标，这样的方法耗时长不说，对于分辨率不一样的目标识别效果也并不好，由此催生了我们的RCNN，在这一讲中我们并不对RCNN的具体算法进行介绍，我们仅仅只讨论这个工具怎么用．　　
在tensorflow中很多模型都已经被实现了，所以我们也不需要再重复造轮子，[具体参考](https://github.com/tensorflow/models.git)这里有使用tensorflow实现的各种深度学习算法，以及论文中出现的比较新的算法，通过tensorflow工具，很方便的去调用，我们针对objecct-detection这个功能进行深入的研究，首先不研究算法，我们先看看他的数据转换的过程：
```python
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import hashlib
import io
import logging
import os

from lxml import etree
import PIL.Image
import tensorflow as tf

from object_detection.utils import dataset_util
from object_detection.utils import label_map_util


flags = tf.app.flags
flags.DEFINE_string('data_dir', '', 'Root directory to raw PASCAL VOC dataset.')
flags.DEFINE_string('set', 'train', 'Convert training set, validation set or '
                    'merged set.')
flags.DEFINE_string('annotations_dir', 'Annotations',
                    '(Relative) path to annotations directory.')
flags.DEFINE_string('year', 'VOC2007', 'Desired challenge year.')
flags.DEFINE_string('output_path', '', 'Path to output TFRecord')
flags.DEFINE_string('label_map_path', 'data/pascal_label_map.pbtxt',
                    'Path to label map proto')
flags.DEFINE_boolean('ignore_difficult_instances', False, 'Whether to ignore '
                     'difficult instances')
FLAGS = flags.FLAGS

SETS = ['train', 'val', 'trainval', 'test']
YEARS = ['VOC2007', 'VOC2012', 'merged']


def dict_to_tf_example(data,
                       dataset_directory,
                       label_map_dict,
                       ignore_difficult_instances=False,
                       image_subdirectory='JPEGImages'):
  img_path = os.path.join(data['folder'], image_subdirectory, data['filename'])
  full_path = os.path.join(dataset_directory, img_path)
  with tf.gfile.GFile(full_path, 'rb') as fid:
    encoded_jpg = fid.read()
  encoded_jpg_io = io.BytesIO(encoded_jpg)
  image = PIL.Image.open(encoded_jpg_io)
  if image.format != 'JPEG':
    raise ValueError('Image format not JPEG')
  key = hashlib.sha256(encoded_jpg).hexdigest()

  width = int(data['size']['width'])
  height = int(data['size']['height'])

  xmin = []
  ymin = []
  xmax = []
  ymax = []
  classes = []
  classes_text = []
  truncated = []
  poses = []
  difficult_obj = []
  for obj in data['object']:
    difficult = bool(int(obj['difficult']))
    if ignore_difficult_instances and difficult:
      continue

    difficult_obj.append(int(difficult))

    xmin.append(float(obj['bndbox']['xmin']) / width)
    ymin.append(float(obj['bndbox']['ymin']) / height)
    xmax.append(float(obj['bndbox']['xmax']) / width)
    ymax.append(float(obj['bndbox']['ymax']) / height)
    classes_text.append(obj['name'].encode('utf8'))
    classes.append(label_map_dict[obj['name']])
    truncated.append(int(obj['truncated']))
    poses.append(obj['pose'].encode('utf8'))

  example = tf.train.Example(features=tf.train.Features(feature={
      'image/height': dataset_util.int64_feature(height),
      'image/width': dataset_util.int64_feature(width),
      'image/filename': dataset_util.bytes_feature(
          data['filename'].encode('utf8')),
      'image/source_id': dataset_util.bytes_feature(
          data['filename'].encode('utf8')),
      'image/key/sha256': dataset_util.bytes_feature(key.encode('utf8')),
      'image/encoded': dataset_util.bytes_feature(encoded_jpg),
      'image/format': dataset_util.bytes_feature('jpeg'.encode('utf8')),
      'image/object/bbox/xmin': dataset_util.float_list_feature(xmin),
      'image/object/bbox/xmax': dataset_util.float_list_feature(xmax),
      'image/object/bbox/ymin': dataset_util.float_list_feature(ymin),
      'image/object/bbox/ymax': dataset_util.float_list_feature(ymax),
      'image/object/class/text': dataset_util.bytes_list_feature(classes_text),
      'image/object/class/label': dataset_util.int64_list_feature(classes),
      'image/object/difficult': dataset_util.int64_list_feature(difficult_obj),
      'image/object/truncated': dataset_util.int64_list_feature(truncated),
      'image/object/view': dataset_util.bytes_list_feature(poses),
  }))
  return example


def main(_):
  if FLAGS.set not in SETS:
    raise ValueError('set must be in : {}'.format(SETS))
  if FLAGS.year not in YEARS:
    raise ValueError('year must be in : {}'.format(YEARS))

  data_dir = FLAGS.data_dir
  years = ['VOC2007', 'VOC2012']
  if FLAGS.year != 'merged':
    years = [FLAGS.year]

  writer = tf.python_io.TFRecordWriter(FLAGS.output_path)

  label_map_dict = label_map_util.get_label_map_dict(FLAGS.label_map_path)

  for year in years:
    logging.info('Reading from PASCAL %s dataset.', year)
    examples_path = os.path.join(data_dir, year, 'ImageSets', 'Main',
                                 'aeroplane_' + FLAGS.set + '.txt')
    annotations_dir = os.path.join(data_dir, year, FLAGS.annotations_dir)
    examples_list = dataset_util.read_examples_list(examples_path)
    for idx, example in enumerate(examples_list):
      if idx % 100 == 0:
        logging.info('On image %d of %d', idx, len(examples_list))
      path = os.path.join(annotations_dir, example + '.xml')
      with tf.gfile.GFile(path, 'r') as fid:
        xml_str = fid.read()
      xml = etree.fromstring(xml_str)
      data = dataset_util.recursive_parse_xml_to_dict(xml)['annotation']

      tf_example = dict_to_tf_example(data, FLAGS.data_dir, label_map_dict,
                                      FLAGS.ignore_difficult_instances)
      writer.write(tf_example.SerializeToString())

  writer.close()


if __name__ == '__main__':
  tf.app.run()
```
上面的代码是将PASCAL数据转换为tensorflow输入数据的方法，实际上PASCAL数据就是一个XML数据，数据中有一些标识，这样构成了影像的一个标识集，然后每一个标识的name对应一个id号，这样我们就可以获取每一个选出来的区域对应的id了，XML数据如下：
```XML
<?xml version="1.0" ?>
<annotation>
	<folder>Img</folder>
	<filename>DJI_0740</filename>
	<path>/home/wuwei/Data/UAVData/8/Img/DJI_0740.JPG</path>
	<source>
		<database>Unknown</database>
	</source>
	<size>
		<width>4000</width>
		<height>3000</height>
		<depth>3</depth>
	</size>
	<segmented>0</segmented>
	<object>
		<name>dog</name>
		<pose>Unspecified</pose>
		<truncated>0</truncated>
		<difficult>0</difficult>
		<bndbox>
			<xmin>1084</xmin>
			<ymin>1231</ymin>
			<xmax>1928</xmax>
			<ymax>1668</ymax>
		</bndbox>
	</object>
	<object>
		<name>person</name>
		<pose>Unspecified</pose>
		<truncated>0</truncated>
		<difficult>0</difficult>
		<bndbox>
			<xmin>1690</xmin>
			<ymin>1443</ymin>
			<xmax>2550</xmax>
			<ymax>2140</ymax>
		</bndbox>
	</object>
	<object>
		<name>person</name>
		<pose>Unspecified</pose>
		<truncated>0</truncated>
		<difficult>0</difficult>
		<bndbox>
			<xmin>2068</xmin>
			<ymin>771</ymin>
			<xmax>2700</xmax>
			<ymax>1115</ymax>
		</bndbox>
	</object>
	<object>
		<name>cat</name>
		<pose>Unspecified</pose>
		<truncated>0</truncated>
		<difficult>0</difficult>
		<bndbox>
			<xmin>681</xmin>
			<ymin>1846</ymin>
			<xmax>1234</xmax>
			<ymax>2300</ymax>
		</bndbox>
	</object>
	<object>
		<name>car</name>
		<pose>Unspecified</pose>
		<truncated>0</truncated>
		<difficult>0</difficult>
		<bndbox>
			<xmin>518</xmin>
			<ymin>968</ymin>
			<xmax>1381</xmax>
			<ymax>1356</ymax>
		</bndbox>
	</object>
</annotation>
```
上面的XML就是一个典型的PASCAl数据XML示例，好了现在我们分析一下这个XML,在一个annotation标签之间包含了一下内容：影像文件夹名称，影像名称，影像路径，影像的大小，以及在影像上标记出来的多个object,每一个object包含了名称，姿态，还有两个字段不知道什么意思，搞不清楚也没关系，反正也用不上，然后是比较重要的box的信息，实际上这个信息就是目标在影像上的位置，有了以上信息做铺垫我们就可以实现标记和训练自己的训练数据了，我们这里再介绍一个工具：[传送门](https://github.com/tzutalin/labelImg.git)代码是用python写的，用到了pyQt库，使用方法非常简单，在这里就不多做说明，通过这个工具标记后就可以获取到上面类似的XML数据，然后我们的重点就是将这个数据转换为TFRecord数据，具体的转换方法为：
```python
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import hashlib
import io
import logging
import os

from lxml import etree
import PIL.Image
import tensorflow as tf

from object_detection.utils import dataset_util
from object_detection.utils import label_map_util

flags = tf.app.flags
flags.DEFINE_string('output_path', '/home/wuwei/Data/MachineLearning/out', 'Path to output TFRecord')
flags.DEFINE_string('label_map_path', 'data/pascal_label_map.pbtxt','Path to label map proto')
FLAGS = flags.FLAGS

def dict_to_tf_example(data,label_map_dict):
    img_path = data['path']
    with tf.gfile.GFile(img_path, 'rb') as fid:
        encoded_jpg = fid.read()
    xmin = []
    ymin = []
    xmax = []
    ymax = []
    classes = []
    classes_text = []
    width = int(data['size']['width'])
    height = int(data['size']['height'])
    for obj in data['object']:
        xmin.append(float(obj['bndbox']['xmin']) / width)
        ymin.append(float(obj['bndbox']['ymin']) / height)
        xmax.append(float(obj['bndbox']['xmax']) / width)
        ymax.append(float(obj['bndbox']['ymax']) / height)
        classes_text.append(obj['name'].encode('utf8'))
        classes.append(label_map_dict[obj['name']])

    tf_example = tf.train.Example(features=tf.train.Features(feature={
        'image/height': dataset_util.int64_feature(height),
        'image/width': dataset_util.int64_feature(width),
        'image/filename': dataset_util.bytes_feature(data['filename'].encode('utf8')),
        'image/source_id': dataset_util.bytes_feature(data['filename'].encode('utf8')),
        'image/encoded': dataset_util.bytes_feature(encoded_jpg),
        'image/format': dataset_util.bytes_feature('jpeg'.encode('utf8')),
        'image/object/bbox/xmin': dataset_util.float_list_feature(xmin),
        'image/object/bbox/xmax': dataset_util.float_list_feature(xmax),
        'image/object/bbox/ymin': dataset_util.float_list_feature(ymin),
        'image/object/bbox/ymax': dataset_util.float_list_feature(ymax),
        'image/object/class/text': dataset_util.bytes_list_feature(classes_text),
        'image/object/class/label': dataset_util.int64_list_feature(classes),
    }))
    return tf_example

def main(_):
    writer = tf.python_io.TFRecordWriter(FLAGS.output_path)
    test_xml='/home/wuwei/Data/MachineLearning/8/DJI_0740.xml'
    test_label='/home/wuwei/Data/MachineLearning/label.pbtxt'
    label_map_dict = label_map_util.get_label_map_dict(test_label)
    with tf.gfile.GFile(test_xml, 'r') as fid:
        xml_str = fid.read()
    xml = etree.fromstring(xml_str)
    data = dataset_util.recursive_parse_xml_to_dict(xml)['annotation']
    tf_example = dict_to_tf_example(data,label_map_dict)
    writer.write(tf_example.SerializeToString())
    writer.close()

if __name__ == '__main__':
    tf.app.run()
```
ok,简单分析以上代码，读取XML,然后通过PIL模块解析影像，将影像读取为数据存入，然后得到每一个objecct,最后写入文件中，整体过程就是这个样子，很简单，[代码传送门](https://github.com/RemoteSensingFrank/tensorflow-learn.git)大家随便下随便用，有什么问题可以一起改进
