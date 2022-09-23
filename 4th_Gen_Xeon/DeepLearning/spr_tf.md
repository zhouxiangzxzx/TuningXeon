# Using TensorFlow*

Starting with TensorFlow Version 2.9, the performance improvements delivered by the [Intel® oneAPI Deep Neural Network Library (oneDNN)](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onednn.html) are turned on by default. This applies to all Linux x86 packages and CPUs with neural-network-focused hardware features such as AVX512_VNNI, AVX512_BF16, and AMX vector and matrix extensions on 2nd, 3rd, and 4th Generation Intel® Xeon® Scalable processors.  These extensions maximize AI performance through efficient compute resource usage, improved cache utilization, and efficient numeric formatting.   The optimizations enabled by oneDNN accelerate key performance-intensive operations such as convolution, matrix multiplication, and batch normalization.  Users have experienced up to [3 times the performance improvements](https://medium.com/intel-analytics-software/leverage-intel-deep-learning-optimizations-in-tensorflow-129faa80ee07) compared to versions without oneDNN acceleration.  Every project is unique and performance improvements may vary.

## Install the latest version of TensorFlow* with pip

Please refer to the [Installation Guide](https://www.tensorflow.org/install/pip) to install the latest stable version of TensorFlow*.

## Enabling BFLOAT16
Intel® 4th Generation Intel® Xeon® Scalable Processors support accelerating AI inference by using low precision data types such as BFloat16 and INT8 based on the Intel® Deep Learning Boost and Intel® Advanced Matrix Extension(AMX). There are several instructions such as AMX_BF16, AMX_INT8, AVX512_BF16, AVX512_VNNI to accelerate AI models.

### For Inference

For a pre-trained FP32 model (resnet50 from TensorFlow Hub as an example below):

1. Need to convert the model to a mixed precision model using [Intel® Neural Compressor](https://intel.github.io/neural-compressor) 

   - Install Intel® Neural Compressor in TensorFlow python environment
   
     ```
     pip install neural-compressor
     ```
     
   - Run below code to convert the model
     
     ```
     import os
     import tensorflow_hub as tf_hub
     from neural_compressor.experimental import MixedPrecision
     
     os.environ["TFHUB_CACHE_DIR"] = 'tfhub_models'
     model = tf_hub.KerasLayer('https://tfhub.dev/google/imagenet/resnet_v1_50/classification/5')
     
     converter = MixedPrecision()
     converter.precisions = 'bf16'
     converter.model = 'tfhub_models/817c3c3182a818a20e7f49d12d803077e4019a1b'
     optimized_model = converter()
     optimized_model.save('resnet50_v1_50_bf16.pb')
     ```

2. Run inference with export ONEDNN_VERBOSE=1, you should be able to see AVX512_BF16 and AMX_BF16 instructions are enabled.

   - Save below code as tf_sample_inf.py
   
     ```
     import tensorflow as tf

     model = tf.saved_model.load('resnet50_v1_50_bf16.pb')
     model.signatures["serving_default"](tf.random.uniform((1, 224, 224, 3)))
     ```
     
   - Run inference script in terminal
   
     ```
     export ONEDNN_VERBOSE=1
     python tf_sample_inf.py
     ```
     
   - Check the result
   
     ```
     onednn_verbose,info,oneDNN v2.6.0 (commit N/A)
     onednn_verbose,info,cpu,runtime:threadpool,nthr:56
     onednn_verbose,info,cpu,isa:Intel AVX-512 with Intel DL Boost and bfloat16 support and Intel AMX with bfloat16 and 8-bit integer support
     onednn_verbose,info,gpu,runtime:none
     onednn_verbose,info,prim_template:operation,engine,primitive,implementation,prop_kind,memory_descriptors,attributes,auxiliary,problem_desc,exec_time
     onednn_verbose,exec,cpu,reorder,jit:uni,undef,src_bf16::blocked:cdba:f0 dst_bf16::blocked:Adcb16a:f0,,,64x3x7x7,0.228027
     onednn_verbose,exec,cpu,convolution,jit:avx512_core_amx_bf16,forward_training,src_bf16::blocked:acdb:f0 wei_bf16::blocked:Adcb16a:f0 bia_undef::undef::f0 dst_bf16::blocked:acdb:f0,attr-scratchpad:user ,alg:convolution_direct,mb1_ic3oc64_ih230oh112kh7sh2dh0ph0_iw230ow112kw7sw2dw0pw0,0.883057
     onednn_verbose,exec,cpu,batch_normalization,bnorm_tbb_jit:avx512_core,forward_inference,data_f32::blocked:acdb:f0 diff_undef::undef::f0,,flags:GSR,mb1ic64ih112iw112,0.307129
     onednn_verbose,exec,cpu,pooling_v2,jit:avx512_core_bf16,forward_training,src_bf16::blocked:acdb:f0 dst_bf16::blocked:acdb:f0 ws_u8::blocked:acdb:f0,,alg:pooling_max,mb1ic64_ih112oh56kh3sh2dh0ph0_iw112ow56kw3sw2dw0pw0,0.305908
     ```

### For Training

You can also train a mixed precision model with bfloat16 using TensorFlow Keras API

```
import tensorflow as tf
from keras.utils import np_utils
from tensorflow.keras import mixed_precision

mixed_precision.set_global_policy('mixed_bfloat16')

# load data
cifar10 = tf.keras.datasets.cifar10
(x_train, y_train), (x_test, y_test) = cifar10.load_data()
num_classes = 10

# pre-process
x_train, x_test = x_train/255.0, x_test/255.0
y_train = np_utils.to_categorical(y_train, num_classes)
y_test = np_utils.to_categorical(y_test, num_classes)

# build model
feature_extractor_layer = tf.keras.applications.ResNet50(include_top=False, weights='imagenet')
feature_extractor_layer.trainable = False
model = tf.keras.Sequential([
    tf.keras.layers.Input(shape=(32, 32, 3)),
    feature_extractor_layer,
    tf.keras.layers.Flatten(),
    tf.keras.layers.Dense(1024, activation='relu'),
    tf.keras.layers.Dropout(0.2),
    tf.keras.layers.Dense(num_classes, activation='softmax')
])
model.compile(
  optimizer=tf.keras.optimizers.Adam(),
  loss=tf.keras.losses.CategoricalCrossentropy(),
  metrics=['acc'])

# train model
model.fit(x_train, y_train,
    batch_size = 128,
    validation_data=(x_test, y_test), 
    epochs=1)

model.save('resnet_bf16_model')
```

Run inference with export ONEDNN_VERBOSE=1

```
import tensorflow as tf

model = tf.saved_model.load('resnet_bf16_model')
model.signatures["serving_default"](tf.random.uniform((1, 32, 32, 3)))
```
You should be able to see avx512_core_bf16 kernels
```
onednn_verbose,info,oneDNN v2.6.0 (commit N/A)
onednn_verbose,info,cpu,runtime:threadpool,nthr:56
onednn_verbose,info,cpu,isa:Intel AVX-512 with Intel DL Boost and bfloat16 support and Intel AMX with bfloat16 and 8-bit integer support
onednn_verbose,info,gpu,runtime:none
onednn_verbose,info,prim_template:operation,engine,primitive,implementation,prop_kind,memory_descriptors,attributes,auxiliary,problem_desc,exec_time
onednn_verbose,exec,cpu,reorder,jit:uni,undef,src_bf16::blocked:cdba:f0 dst_bf16::blocked:Adcb16a:f0,,,64x3x7x7,0.199951
onednn_verbose,exec,cpu,convolution,jit:avx512_core_amx_bf16,forward_training,src_bf16::blocked:acdb:f0 wei_bf16::blocked:Adcb16a:f0 bia_bf16::blocked:a:f0 dst_bf16::blocked:acdb:f0,attr-scratchpad:user ,alg:convolution_direct,mb1_ic3oc64_ih32oh16kh7sh2dh0ph3_iw32ow16kw7sw2dw0pw3,0.551025
onednn_verbose,exec,cpu,batch_normalization,bnorm_tbb_jit:avx512_core_bf16,forward_inference,data_bf16::blocked:acdb:f0 diff_undef::undef::f0,,flags:GSR,mb1ic64ih16iw16,0.196045
onednn_verbose,exec,cpu,pooling_v2,jit:avx512_core_bf16,forward_training,src_bf16::blocked:acdb:f0 dst_bf16::blocked:acdb:f0 ws_u8::blocked:acdb:f0,,alg:pooling_max,mb1ic64_ih18oh8kh3sh2dh0ph0_iw18ow8kw3sw2dw0pw0,0.14209
```

## Enabling INT8

For INT8 optimization, use [Intel® Neural Compressor](https://intel.github.io/neural-compressor) to quantize the model first, then run inference.  This has improved performance on the 4th Generation Intel® Xeon® Scalable Processors.
