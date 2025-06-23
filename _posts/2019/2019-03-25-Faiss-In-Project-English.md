---
layout: post
title: Faiss Practice
date: 2019-03-25 21:44:04
author: admin
comments: true
categories: [Faiss]
tags: [Faiss, Image Search]
---

I started to learn Faiss in September 2018. 
Although there was a lot of confusion at the beginning, it was finally officially used in the project after two months of study. 

Here is a summary.  [中文版](../Faiss-In-Project/).

<!-- more -->

---

* 目录
{:toc}
---

# Ask Again

## 1. What is Faiss ?

Regardless of the specific definition, the best way to understand  new things is to compare things that you are familiar with.

For example, Faiss can be analogized to a **database** that can be **indexed**.

What is the purpose of the **index**? Faster reading. What can the database do? Add, delete and change. What is stored in the database? It's usually a lot of records, but for Faiss it's a huge number of vectors.

The difference between Faiss and the database is that the data in Faiss is all Index.

## 2. What is the role of Index in Faiss?

We can still compare that with database. For to query our data faster, we can index the first letter like a dictionary. Or use the inverted index like many search engines.

Different indexing methods have different advantages and disadvantages. 

Faiss has implemented many type indexes. If you just want to use them, you can temporarily ignore their implementation principles, just need to understand their own characteristics and their own use scenarios. Detail you can refer to [this](https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index).

---

# My Implementation

## 0. Environment

No doubt using docker

Faiss docker：_https://hub.docker.com/r/waltyou/faiss-api-service/_


## 1. Extract Image Feature

There are too many image feature extraction algorithms for us to choose.

**But this step is actually not concerned with Faiss, it only cares about building the index.**

I use the **SIFT** in my project. This algorithm can find a lot of feature points from a image, each feature point has a corresponding 128-dimensional vector. These vectors are the feature vectors we want.

```python
// create a opencv sift extractor
def get_sift():
    return cv2.xfeatures2d.SIFT_create(nfeatures=NUM_FEATURES, nOctaveLayers=3, contrastThreshold=0.04, edgeThreshold=10, sigma=1.6)

// calculate sift
def calc_sift(sift, image_file):
    if not os.path.isfile(image_file):
        logging.error('Image:{} does not exist'.format(image_file))
        return -1, None

    try:
        image_o = cv2.imread(image_file)
    except:
        logging.error('Open Image:{} failed'.format(image_file))
        return -1, None

    if image_o is None:
        logging.error('Open Image:{} failed'.format(image_file))
        return -1, None

    image = cv2.resize(image_o, (NOR_X, NOR_Y))
    if image.ndim == 2:
        gray_image = image
    else:
        gray_image = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

    kp, des = sift.detectAndCompute(gray_image, None)

    sift_feature = np.matrix(des)
    return 0, sift_feature
```

#### Attention

There have a problem when using SIFT, that is a picture can extract many feature vectors. So the picture corresponding vector is a one-to-many relationship, but the basic unit of the Faiss search is a single vector (of course, it can input multiple vectors at a time) .

This means that Faiss enters a vector x by default, returning the k vectors that are most similar to x. 

To solve this problem, you can assign multiple ids to multiple vectors of an image when building a Faiss index.
In this way, after searching with multiple vectors of a picture, in the returned result, only the number of times the associated id appears can be counted, and the similarity level can be obtained. This can be seen later in the code.

If you are familiar with neural networks such as CNN, you can also use them to generate a single high-dimensional vector for a single image.

## 2. Build Index

I use the most basic Index type: "IDMap, Flat". You can choose the type you need.

```python
# prepare index
dimensions = 128
INDEX_KEY = "IDMap,Flat"
index = faiss.index_factory(dimensions, INDEX_KEY)
if USE_GPU:
    res = faiss.StandardGpuResources()
    index = faiss.index_cpu_to_gpu(res, 0, index)

id = 0
index_dict = {}
for file_name in image_list:
    ret, sift_feature = calc_sift(sift, file_name)
    if ret == 0 and sift_feature.any():
        # record id and path
        index_dict.update({id: (file_name, sift_feature)})
        ids_list = np.linspace(ids_count, ids_count, num=sift_feature.shape[0], dtype="int64")
        id += 1
        index.add_with_ids(sift_feature, ids_list)

```

We can save the `index` and `index_dict` as file in the end.

```python
# save index
faiss.write_index(index, index_path)

# save ids
with open(ids_vectors_path, 'wb+') as f:
    pickle.dump(index_dict, f, True)
f.close()
```
> If you choose the type of Index you need to train, train it first and then add the vector.



## 3. Search Index

```python
scores, neighbors = index.search(siftfeature, k=topN)
```

## 4. Build API Service

Mainly refer to "plippe/faiss-web-service", you can find in github.

---

# Summary

The complete code can be seen on github： [waltyou/faiss-web-service](https://github.com/waltyou/faiss-web-service)。

