---
tags: [moc, cv]
---

# 11 — COMPUTER VISION

> Teaching machines to interpret images.

**Why it matters:** An image is just a tensor of numbers. Everything in vision is operations on that tensor.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Fundamentals

Images as tensors `(B, C, H, W)` · pixels · channels · **convolution** · kernels · feature maps · pooling

> An image is `(batch, channels, height, width)` in PyTorch — **channels before height/width**. TensorFlow puts channels last. This catches everyone once.

## Tasks

| Task | Output | Example |
|---|---|---|
| **Classification** | One label | "This is a turbine blade" |
| **Object detection** | Boxes + labels | "Crack at (x,y,w,h)" |
| **Segmentation** | Per-pixel label | Exact crack outline |
| **Tracking** | Object identity over time | Follow across frames |
| **Pose estimation** | Keypoints | Robot arm joint positions |
| **Optical flow** | Per-pixel motion | Velocity field |

## Architectures

CNNs ([[CNNs and transfer learning]]) · ResNet · EfficientNet · U-Net (segmentation) · YOLO (detection) · Vision Transformers

## The practical rule

**Always start from a pretrained backbone.** Training from scratch needs millions of images. Fine-tuning needs hundreds. See [[CNNs and transfer learning]].

## Where it's used here
[[21 — ROBOTICS]] — perception · [[23 — AEROSPACE]] — inspection · quality control in manufacturing

## Related
[[09 — DEEP LEARNING]] · [[10 — PYTORCH]] · [[21 — ROBOTICS]]
