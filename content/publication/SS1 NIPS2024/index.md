---
title: 'SS1: Accelerating Inference with Fast and Expressive Sketch Structured Transform'

# Authors
# If you created a profile for a user (e.g. the default `admin` user), write the username (folder name) here
# and it will be replaced with their full name and linked to their profile.
authors:
  - Kimia Saedi
  - Aditya Desai
  - Apoorv Walia
  - Jihyeong Lee
  - Keren Zhou
  - Anshumali Shrivastava

# Author notes (optional)
# author_notes:
#   - 'Equal contribution'
#   - 'Equal contribution'

date: '2024-12-15T05:30:00Z'
doi: ''

# Schedule page publish date (NOT publication's date).
publishDate: '2025-11-03T00:00:00Z'

# Publication type.
# Accepts a single type but formatted as a YAML list (for Hugo requirements).
# Enter a publication type from the CSL standard.
publication_types: ['paper-conference']

# Publication name and optional abbreviated publication name.
publication: In The Thirty-eighth Annual Conference on Neural Information Processing Systems
publication_short: In *NeurIPS 2024*

abstract: Tensor multiplication with learned weight matrices is the fundamental building block in deep learning models. These matrices can often be sparsified, decomposed, quantized, or subjected to random parameter sharing without losing accuracy, suggesting the possibility of more efficient transforms. Although many variants of weight matrices exist, unstructured ones are incompatible with modern hardware, slowing inference and training. On the other hand, structured variants often limit expressivity or fail to deliver the promised latency benefits. We present Sketch Structured Transform (SS1), an expressive and GPU-friendly operator that accelerates inference. SS1 leverages parameter sharing in a random yet structured manner to reduce computation while retraining the rich expressive nature of parameter sharing. We confirm empirically that SS1 offers better quality-efficiency tradeoffs than competing variants. Interestingly SS1 can be combined with Quantization to achieve gains unattainable by either method alone, a finding we justify via theoretical analysis. The analysis may be of independent interest.Moreover, existing pre-trained models can be projected onto SS1 and finetuned for efficient deployment. Surprisingly, these projected models can perform reasonably well even without finetuning. Our experiments highlight various applications of the SS1:(a) Training GPT2 and DLRM models from scratch for faster inference. (b) Finetuning projected BERT models for 1.31× faster inference while maintaining GLUE scores. (c) Proof of concept with Llama-3-8b, showing 1.11× faster wall clock inference using projected SS1 layers without finetuning. We open source our code :https://github.com/apd10/Sketch-Structured-Linear/

# Summary. An optional shortened abstract.
# summary: Lorem ipsum dolor sit amet, consectetur adipiscing elit. Duis posuere tellus ac convallis placerat. Proin tincidunt magna sed ex sollicitudin condimentum.

tags: []

# Display this page in the Featured widget?
featured: false

# Custom links (uncomment lines below)
# links:
# - name: Custom Link
#   url: http://example.org

url_pdf: 'https://proceedings.neurips.cc/paper_files/paper/2024/file/57ef0373c890b30407eadfe6e06c8c84-Paper-Conference.pdf'
# url_code: 'https://github.com/HugoBlox/hugo-blox-builder'
# url_dataset: 'https://github.com/HugoBlox/hugo-blox-builder'
# url_poster: ''
# url_project: ''
# url_slides: 'uploads/MLSH2024.pdf'
# url_source: 'https://github.com/HugoBlox/hugo-blox-builder'
# url_video: 'https://youtube.com'

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# image:
#   caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/pLCdAaMFLTE)'
#   focal_point: ''
#   preview_only: false

# Associated Projects (optional).
#   Associate this publication with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `internal-project` references `content/project/internal-project/index.md`.
#   Otherwise, set `projects: []`.
# projects:
#   - example

# Slides (optional).
#   Associate this publication with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides: "example"` references `content/slides/example/index.md`.
#   Otherwise, set `slides: ""`.
# slides: example
---

<!-- {{% callout note %}}
Click the _Cite_ button above to demo the feature to enable visitors to import publication metadata into their reference management software.
{{% /callout %}}

{{% callout note %}}
Create your slides in Markdown - click the _Slides_ button to check out the example.
{{% /callout %}} -->

<!-- Add the publication's **full text** or **supplementary notes** here. You can use rich formatting such as including [code, math, and images](https://docs.hugoblox.com/content/writing-markdown-latex/). -->
