# GenshinCLIP
Simple open-sourced CLIP models fine-tuned on Genshin Impact's image-text pairs.

The models are far from being perfect, but could still offer some better text-image matching performance in some Genshin Impact scenarios.

#### Installtation
```bash
sudo apt-get update && sudo apt-get install cbm ffmpeg git-lfs 

conda create -n py310 python=3.10 && conda activate py310
pip install ipykernel
python -m ipykernel install --user --name py310 --display-name "py310"

pip install open-clip-torch torch torchvision
```

#### 5.0 QingCe Demp
```python
import torch
import torch.nn.functional as F
from PIL import Image
import requests
from open_clip import create_model_from_pretrained, get_tokenizer

def preprocess_text(string):
    return "Genshin Impact\n" + string

device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")

# load checkpoint from local path
# model_path = "path/to/open_clip_pytorch_model.bin"
# model_name = "ViT-SO400M-14-SigLIP-384"
# model, preprocess = create_model_from_pretrained(model_name=model_name, pretrained=model_path, device=device)
# tokenizer = get_tokenizer(model_name)

# or load from hub
model, preprocess = create_model_from_pretrained('hf-hub:mrzjy/GenshinImpact-5.0-ViT-SO400M-14-SigLIP-384', device=device)
tokenizer = get_tokenizer('hf-hub:mrzjy/GenshinImpact-5.0-ViT-SO400M-14-SigLIP-384')

# image
image_url = "https://static.wikia.nocookie.net/gensin-impact/images/3/33/Qingce_Village.png"
image = Image.open(requests.get(image_url, stream=True).raw)
image = preprocess(image).unsqueeze(0).to(device)

# text choices
labels = [
    "This is an area of Liyue",
    "This is an area of Mondstadt",
    "This is an area of Sumeru",
    "This is Qingce Village"
]
labels = [preprocess_text(l) for l in labels]
text = tokenizer(labels, context_length=model.context_length).to(device)
with torch.autocast(device_type=device.type):
    with torch.no_grad():
        image_features = model.encode_image(image)
        text_features = model.encode_text(text)
        image_features = F.normalize(image_features, dim=-1)
        image_features = F.normalize(image_features, dim=-1)
        text_features = F.normalize(text_features, dim=-1)
        text_probs = torch.sigmoid(image_features @ text_features.T * model.logit_scale.exp() + model.logit_bias)
        scores = [f"{s:.3f}" for i, s in enumerate(text_probs.tolist()[0])]
        print(scores)

```

## Model Link

| Dataset          | Model Name                                    | Link                                                                                      | Checkpoint Size | Val Loss |
|------------------|-----------------------------------------------|-------------------------------------------------------------------------------------------|-----------------|----------|
| Game Version 5.0 | GenshinImpact-5.0-ViT-SO400M-14-SigLIP-384    | [Huggingface](https://huggingface.co/mrzjy/GenshinImpact-5.0-ViT-SO400M-14-SigLIP-384)    | 3.51 GB         | 0.519    |
| Game Version 4.x | GenshinImpact-CLIP-ViT-B-16-laion2B-s34B-b88K | [Huggingface](https://huggingface.co/mrzjy/GenshinImpact-CLIP-ViT-B-16-laion2B-s34B-b88K) | 0.59 GB         | 1.152    |
| Game Version 4.x | GenshinImpact-ViT-SO400M-14-SigLIP-384        | [Huggingface](https://huggingface.co/mrzjy/GenshinImpact-ViT-SO400M-14-SigLIP-384)        | 3.51 GB         | 0.362    |

## Case Study

- For GenshinImpact-ViT-SO400M-14-SigLIP-384

| Case | Image                                                                                                                                              | Multiple Choices                                                                                                                        | CLIPScore<br/>(After Finetune)                   | CLIPScore<br/>(Before Finetune)                                     |
|------|----------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|---------------------------------------------------------------------|
| 1    | <img src="https://static.wikia.nocookie.net/gensin-impact/images/2/24/Ganyu_Card.png/revision/latest?cb=20230519012433" height="64">               | 1) This is an image of Kuki Shinobu<br/>2) This is an image of Ganyu<br/>3) This is an image of Keqing<br/>4) This is an image of Liyue | 1) 0.000<br>2) **0.438**<br>3) 0.000<br>4) 0.000 | 1) **1.207e-06**<br/>2) 4.144e-08<br/>3) 1.201e-07<br/>4) 2.212e-08 |
| 2    | <img src="https://static.wikia.nocookie.net/gensin-impact/images/3/33/Qingce_Village.png/revision/latest?cb=20220626161951" height="64">           | 1) This is an area of Liyue<br/>2) This is an area of Mondstadt<br/>3) This is an area of Sumeru<br/>4) This is Qingce Village          | 1) 0.016<br>2) 0.000<br>3) 0.001<br>4) **0.233** | 1) 0.015<br/>2) 0.003<br/>3) 0.009<br/>4) **0.377**                 |
| 3    | <img src="https://static.wikia.nocookie.net/gensin-impact/images/0/0c/Enemy_Boreas.png/revision/latest?cb=20210426192800" height="64">             | 1) This is Andrius Wolf of the North<br/>2) This is Stormterror Dvalin<br/>3) This is Guardian of Apep's Oasis                          | 1) 1.000<br>2) 0.000<br>3) 0.000                 | 1) **0.001**<br/>2) 0.000<br/>3) 0.000                              |
| 4    | <img src="https://static.wikia.nocookie.net/gensin-impact/images/2/24/Item_Amakumo_Fruit_Wild.png/revision/latest?cb=20220601235905" height="64">  | 1) This is Amakumo Fruit<br/>2) This is Padisarahs<br/>3) This is Naku Weeds                                                            | 1) 0.000<br>2) 0.000<br>3) **0.024**             | 1) 9.425e-07<br/>2) 1.207e-06<br/>3) **1.669e-05**                  |

Note: Case 4 is a bad case. (Correct answer is Amakumo Fruit)


## Intended uses & limitations

You can use the raw model for tasks like zero-shot image classification and image-text retrieval.

### How to use (With OpenCLIP)

Here is how to use this model to perform zero-shot image classification:

```python
import torch
import torch.nn.functional as F
from PIL import Image
import requests
from open_clip import create_model_from_pretrained, get_tokenizer

def preprocess_text(string):
    return "Genshin Impact\n" + string

device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")

# load checkpoint from local path
# model_path = "path/to/open_clip_pytorch_model.bin"
# model_name = "ViT-SO400M-14-SigLIP-384"
# model, preprocess = create_model_from_pretrained(model_name=model_name, pretrained=model_path, device=device)
# tokenizer = get_tokenizer(model_name)

# or load from hub
model, preprocess = create_model_from_pretrained('hf-hub:mrzjy/GenshinImpact-ViT-SO400M-14-SigLIP-384')
tokenizer = get_tokenizer('hf-hub:mrzjy/GenshinImpact-ViT-SO400M-14-SigLIP-384')

# image
image_url = "https://static.wikia.nocookie.net/gensin-impact/images/3/33/Qingce_Village.png"
image = Image.open(requests.get(image_url, stream=True).raw)
image = preprocess(image).unsqueeze(0).to(device)

# text choices
labels = [
    "This is an area of Liyue",
    "This is an area of Mondstadt",
    "This is an area of Sumeru",
    "This is Qingce Village"
]
labels = [preprocess_text(l) for l in labels]
text = tokenizer(labels, context_length=model.context_length).to(device)
with torch.autocast(device_type=device.type):
    with torch.no_grad():
        image_features = model.encode_image(image)
        text_features = model.encode_text(text)
        image_features = F.normalize(image_features, dim=-1)
        image_features = F.normalize(image_features, dim=-1)
        text_features = F.normalize(text_features, dim=-1)
        text_probs = torch.sigmoid(image_features @ text_features.T * model.logit_scale.exp() + model.logit_bias)
        scores = [f"{s:.3f}" for i, s in enumerate(text_probs.tolist()[0])]
        print(scores)  # [0.016, 0.000, 0.001, 0.233]
```

## Model Card
### SigLIP for GenshinImpact

[SigLIP model](https://huggingface.co/timm/ViT-SO400M-14-SigLIP-384) further fine-tuned on 15k Genshin Impact English text-image pairs at resolution 384x384.

### Training data description

All the images and texts are crawled from [Genshin Fandom Wiki](https://genshin-impact.fandom.com/wiki) and are manually parsed to form text-image pairs.

**Image Processing:**
- Size: Resize all images to 384x384 pixels to match the original model training settings.
- Format: Accept images in PNG or GIF format. For GIFs, extract a random frame to create a static image for text-image pairs.

**Text Processing:**
- Source: Text can be from the simple caption attribute of an HTML `<img>` tag or specified web content.
- Format: Prepend all texts with "Genshin Impact" along with some simple template to form natural language sentences.

For example, here are some training image-text pairs:

| Image                                                                | Text                                                                                                                                                                                                                                          |
|----------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| <img src="img/9361c6fe-dabe-43b3-ae76-9731e4b73e1b.png" height="64"> | Genshin Impact<br/>This is Tainted Water-Splitting Phantasm.<br/>Tainted Hydro Phantasm Idle Waving 2                                                                                                                                         |
| <img src="img/911d0f58-dc35-4dfb-9afc-79a2d582881b.png" height="64"> | Genshin Impact<br/>This is Annihilation Specialist Mek.<br/>Although the Fontaine Research Institute of Kinetic Energy Engineering... ..., state apparatus enforcing the monopoly on violence.                                                |
| <img src="img/bd94b015-153f-41de-b8d4-049ba29481e3.png" height="64"> | Genshin Impact<br/>Collei<br/>Expressions<br/>a Smiley expression                                                                                                                                                                             |
| <img src="img/36a956e1-e6af-43bc-a71b-a893fe1bcf9e.png" height="64"> | Genshin Impact<br/>This is the map of viewpoint Nine Pillars of Peace in Cuijue Slope, Minlin, Liyue<br/>Nine shackles of stone were said to have been laid down deep in the valleys of Cuijue Slope to drive off evil and cleanse the world. |

**Data Distribution:**

| 5.0                                                    | 4.x                                                |
|--------------------------------------------------------|----------------------------------------------------|
| <img src="img/data_distribution_5.0.png" height="196"> | <img src="img/data_distribution.png" height="196"> |


**Validation Loss Curve**

![loss_curve.png](img%2Floss_curve.png)
