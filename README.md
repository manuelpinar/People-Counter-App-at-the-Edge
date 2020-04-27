# intel-openvino-people-counter
Project Write-Up

Project Notes with useful commands can be found [here](NOTES.md)

## Model Selection 
Models were selected from the [Tensorflow Object Detection Model Zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md). The following two model were tried:

* [ssd_inception_v2_coco](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz)
* [faster_rcnn_inception_v2_coco](http://download.tensorflow.org/models/object_detection/faster_rcnn_inception_v2_coco_2018_01_28.tar.gz)

The SSD model had good latency (~155 microseconds per frame), but lacked accuracy and failed to detect some people in some of the video frames. Faster-RCNN showed higher latency (~889 microseconds per frame), but provided good accuracy required for this project. Correctly detecting a person in the frame is important for detecting him/her enter to and exit from the scene, and thus the duration he/she spent in the scene.

## Explaining Custom Layers

Any layer not listed as supported by OpenVINO is considered custom. The list of supported layers is quite extensive yet in some cases it may not be possible to avoid handling custom layers as they are present in the given model. Some of the potential reasons:
* The model was obtained from an outside source 
* The model or some of its layers were developed as part of research 
* The model has state of the art architecture and is supposed to be used as is without modifications

The process behind converting custom layers involves several options depending of the original framework. With tensorflow the options are:
* Register custom layers as extensions
* Replace sub-graph with different sub-graph
* Offload calculation back to the tensorflow itself

Thankfully, OpenVINO already contains extensions for custom layers used in [Tensorflow Object Detection Model Zoo](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/detection_model_zoo.md). These extensions' configurations are located in `<OpenVINO install dir>/deployment_tools/model_optimizer/extensions/front/tf`. The Model Optimizer option for providing extension config when converting Tensorflow model to OpenVINO IR format is `--tensorflow_use_custom_operations_config`. 

Since two of the TF Object Detection Zoo models were used in this project, two different extensions were used depending on the model:
* [ssd_inception_v2_coco](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz): ssd_v2_support.json
* [faster_rcnn_inception_v2_coco](http://download.tensorflow.org/models/object_detection/faster_rcnn_inception_v2_coco_2018_01_28.tar.gz): faster_rcnn_support.json

The full Model Optimizer command for converting TF model to OpenVINO IR format is given below:
```sh
python /opt/intel/openvino/deployment_tools/model_optimizer/mo.py --input_model faster_rcnn_inception_v2_coco_2018_01_28/frozen_inference_graph.pb --tensorflow_object_detection_api_pipeline_config faster_rcnn_inception_v2_coco_2018_01_28/pipeline.config --reverse_input_channels --tensorflow_use_custom_operations_config /opt/intel/openvino/deployment_tools/model_optimizer/extensions/front/tf/faster_rcnn_support.json
```

The operations in those custom layers are implemented (for CPU) in the following library: `<OpenVINO install dir>/deployment_tools/inference_engine/lib/intel64/libcpu_extension_sse4.so`. This path should be used with `add_extension()` method of the Python API when initializing the network for inference with OpenVINO.


## Comparing Model Performance

Both models performance before and after conversion to Intermediate Representations was measured for inference latency (per frame) and memory utilization. 

| Model/Framework                             | Latency (microseconds)            | Memory (Mb) |
| -----------------------------------         |:---------------------------------:| -------:|
| ssd_inception_v2_coco (plain TF)            | 222                               | 538    |
| ssd_inception_v2_coco (OpenVINO)            | 155                               | 329    |
| faster_rcnn_inception_v2_coco (plain TF)    | 1280                              | 562    |
| faster_rcnn_inception_v2_coco (OpenVINO)    | 889                               | 281    |

Memory usage was measured with `free -m` (simply by calculating the difference between used memory during inference and when inference process was stopped). 

Latencies were collected and averages calculated after video processing. Below are comparative latency graphs:

![Latency Compare](images/latencies_compare.png)

OpenVINO optimized network produced bounding boxes indistiguishable for the human eye from those inferred with pure Tensorflow, with all people correctly identified in the video. So, in this particular case conversion to OpenVINO allowed us to achieve the project's goals with significant latency (1.44 times) and memory (up to 2 times) improvements.


## Potential Model Use Cases

Some of the potential use cases of the people counter app are as follows.

* Retail Shops - count number of customers/visitors in the shop, the duration of their stay and locations they spent most time at. 
    * For those that stayed longer, analyzing locations thay spent most their time at, can give hints to what products make people stay longer (people like them and/or want to know more about them)
    * Check how often they come to the consultant or cashier to understand how many consultants/employees should be in the shop at different times of the day. Can lead to better employees' schedules, allow part-time shifts etc. which would lead to happier employees and optimized salary budget for the employer
    * Analyze traffic of shoppers to know when more items should be in stock. This should greatly reduce amount of times when customer is turned away because item is not currently available, or when items currently not in demand are stacked in unreasonable amounts

* Voting Room - count total number of voters to double-check the voter turnout numbers at the end of the day. This should reduce the possibility for errors or voter fraud at each particular booth.

* Security Cameras - counting the number of people in the scene and raising an alarm/action when above certain number.
    * Raising alarm when detecting any number of people > 0 on the scene, when no person should be allowed
    * Raising alarm when total count of people is above certain number, e.g. 2, when expecting a gardener visit in the morning and Amazon delivery in the afternoon, but no-one else
    * Turn on high resolution recording every time a camera detects a person (or people) in the scene, should be good for recording criminal activity and/or identifying potential intruders, delivery package thieves etc

## End User Considerations

Lighting, model accuracy, and camera focal length/image size have different effects on a deployed edge model. The potential effects of each of these are as follows:

* Lighting - some models may not identify a person if the lighting is inappropriate, e.g. too dim, insufficient light on the face etc.
* Model accuracy - lower accuracy models may detect a person on one frame, and not detect him/her in the next, even though they never left the scene, and then they may detect them again. This will lead to incorrect count of the duration people spent on the scene, and incorrect total count. This is exactly what happened in this project's video with [ssd_inception_v2_coco](http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2018_01_28.tar.gz) model, and the reason why [faster_rcnn_inception_v2_coco](http://download.tensorflow.org/models/object_detection/faster_rcnn_inception_v2_coco_2018_01_28.tar.gz) model was selected.
* Camera focal length/image size - these greatly influence image quality and therefore the ability of the model to detect what it was trained to detect. In those cases when low quality camera cannot be replaced, and the model performs poorly on its footage, knowledge transfer with additional training on images from this particular camera should be considered.


## Application Run

Application is run with the following command. Note: `-video_size 758x432`, 
not the size of original video `768x432` (shall be fixed soon in the UI).
```commandline
python main.py -i resources/Pedestrian_Detect_2_1_1.mp4 -m model/faster_rcnn_inception_v2_coco.xml -l /opt/intel/openvino/deployment_tools/inference_engine/lib/intel64/libcpu_extension_sse4.so -d CPU -pt 0.4 | ffmpeg -v warning -f rawvideo -pixel_format bgr24 -video_size 758x432 -framerate 24 -i - http://0.0.0.0:3004/fac.ffm
```
The above command assumes ffserver, MQTT server, and the UI application are up and running.
For more details, see [Project Notes](NOTES.md)  

Below is the application UI outlook during video processing:

![screenshot-1](images/app-screenshot-1.png)

![screenshot-2](images/app-screenshot-2.png)

After video processing, the average duration is 17 seconds, 
and the total people count is 6:

![screenshot-3](images/app-screenshot-3.png)


