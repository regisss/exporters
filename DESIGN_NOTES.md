# Design notes for Core ML exporters

The design of the Core ML exporter for 🤗 Transformers is based on that of the ONNX exporter. Both are used in the same manner and in some places the code is very similar. However, there are also differences due to the way Core ML works. This file documents the decisions that went into building the Core ML exporter.

## Philosophy

An important goal of Core ML is to make using models completely hands-off. For example, if a model requires an image as input, you can simply give it an image object, without having to preprocess the image first. And if the model is a classifier, the output is simply the winning class label instead of a logits tensor. The Core ML exporter will add extra operations to the beginning and end of the model where possible, so that users of these models do not have to do their own pre- and postprocessing if Core ML can already handle this for them.

The Core ML exporter is built on top of `coremltools`. This library first converts the PyTorch or TensorFlow model into an intermediate representation known as MIL, then performs optimizations on the MIL graph, and finally serializes the result into a `.mlmodel` or `.mlpackage` file (the latter being the preferred format).

Design of the exporter:

- The Core ML conversion process is described by a `CoreMLConfig` object, analogous to `OnnxConfig`.

- In order to distinguish between the `default` task for text models and vision models, the config object has a `modality` property. For text models, the config object is actually of type `CoreMLTextConfig`. For vision models, `CoreMLVisionConfig`.

- The standard `CoreMLConfig` object already chooses appropriate input and output descriptions for most models. Only models that do something different, for example use BGR input images instead of RGB, need to have their own config object.

- If a user wants to change properties of the inputs or outputs (name, description, other settings), they have to subclass the `XYZCoreMLConfig` object and override these methods. Not very convenient, but it's also not something people will need to do a lot (and if they do, it means we made the wrong default choice).

- Where possible, the behavior of the converted model is described by the tokenizer or feature extractor. For example, to use a different input image size, the user would need to create the feature extractor with those settings and use that during the conversion instead of the default feature extractor.

- The `FeaturesManager` code is copied from `transformers.onnx.features` with minimal changes to the logic, only the table with supported models is different (and using `CoreMLConfig` instead of `OnnxConfig`).

Extra stuff the Core ML exporter does:

- For image inputs, mean/std normalization is performed by the Core ML model. Resizing and cropping the image still needs to be done by the user but is usually left to other Apple frameworks such as Vision.

- Tensor inputs may have a different datatype. Specifically, `bool` and `int64` are converted to `int32` inputs, as that is the only integer datatype Core ML can handle.

- Classifier models that output a single prediction for each input example are treated as special by Core ML. These models have two outputs: one with the class label of the best prediction, and another with a dictionary giving the probabilities for all the classes.

- Models that perform classification but do not fit into Core ML's definition of a classifier, for example a semantic segmentation model, have the list of class names added to the model's metadata. Core ML ignores these class names but they can be retrieved by writing a few lines of Swift code.

- Because the goal is to make the converted models as convenient as possible for users, any model that predicts `logits` has the option of applying a softmax, to output probabilities instead of logits. This option is enabled by default for such models. For image segmentation models, there can be two operations inserted: upsampling to the image's original spatial dimensions, followed by an argmax to select the class index for each pixel.

- The exporter may add extra metadata to allow making predictions from Xcode's model previewer.

- Quantization and other optimizations can automatically be applied by `coremltools`, and therefore are part of the Core ML exporting workflow. The user can always make additional changes to the Core ML afterwards using `coremltools`.

Note: Tokenizers are not a built-in feature of Core ML. A model that requires tokenized input must be tokenized by the user themselves. This is outside the scope of the Core ML exporter.

## Supported tasks

The Core ML exporter supports most of the tasks that the ONNX exporter supports, except for:

- `causal-lm` / `AutoModelForCausalLM`
- `seq2seq-lm` / `AutoModelForSeq2SeqLM`
- `image-segmentation` / `AutoModelForImageSegmentation`

(Didn't get around to implementing these yet.)

Tasks that the Core ML exporter supports but the ONNX exporter currently doesn't:

- `next-sentence-prediction`
- `semantic-segmentation`

Tasks that neither of them support right now:

- `AutoModelForAudioClassification`
- `AutoModelForAudioFrameClassification`
- `AutoModelForAudioXVector`
- `AutoModelForCTC`
- `AutoModelForInstanceSegmentation`
- `AutoModelForPreTraining`
- `AutoModelForSpeechSeq2Seq`
- `AutoModelForTableQuestionAnswering`
- `AutoModelForVideoClassification`
- `AutoModelForVision2Seq`
- `AutoModelForVisualQuestionAnswering`
- `AutoModelWithLMHead`
- `...DoubleHeadsModel`
- `...ForImageClassificationWithTeacher`

Tasks that could be improved:

- `object-detection`. If a Core ML model outputs the predicted bounding boxes in a certain manner, the user does not have to do any decoding and can directly use these outputs in their app (through the Vision framework). Currently, the Core ML exporter does not add this extra functionality.

## Missing features

The following are not supported yet but would be useful to add:

- `-with-past` versions of the tasks. This will make models harder to use, since the user will have to keep track of many additional tensors, but it does make inference much faster on sequences.

- Flexible input sizes. Core ML models typically work with fixed input dimensions, but it also supports flexible image sizes and tensor shapes.

- More quantization options. coremltools 6 adds new quantization options for ML Program models, plus options for sparsifying weights.

There are certain models that cannot be converted because of the way they are structured, or due to limitations and bugs in coremltools. Sometimes these can be fixed by making changes to the Transformers code, by implementing missing ops, or by filing bugs against coremltools. Trying to get as many Transformers models to export without issues is a work in progress.

## Assumptions made by the exporter

The Core ML exporter needs to make certain assumptions about the Transformers models. These are:

- A vision `AutoModel` is expected to output hidden states. If there is a second output, this is assumed to be from the pooling layer.

- The input size for a vision model is given by the feature extractor's `crop_size` property if it exists and `do_center_crop` is true, or otherwise by its `size` property.

- The image normalization for a vision model is given by the feature extractor's `image_std` and `image_mean` if it has those, otherwise assume `std = 1/255` and `mean = 0`.

- The `masked-im` task expects a `bool_masked_pos` tensor as the second input. If `bool_masked_pos` is provided, some of these models return the loss value and others don't. If more than one tensor is returned, we assume the first one is the loss and ignore it.

- If text models have two inputs, the second one is the `attention_mask`. If they have three inputs, the third is `token_type_ids`.

- The `object-detection` task outputs logits and boxes (only tested with YOLOS so far).

- If bicubic resizing is used, it gets replaced by bilinear since Core ML doesn't support bicubic. This has a noticeable effect on the predictions, but usually the model is still usable.

## Other remarks

- Just as in the ONNX exporter, the `validate_model_outputs()` function takes an `atol` argument for the absolute tolerance. It might be more appropriate to do this test as `max(abs(coreml - reference)) / max(abs(reference))` to get an error measurement that's relative to the magnitude of the values in the output tensors.

- Image classifier models have the usual `classLabel` and `probabilities` outputs, but also a "hidden" `var_xxx` output with the softmax results. This appears to be a minor bug in the converter; it doesn't hurt anything to keep this extra output.