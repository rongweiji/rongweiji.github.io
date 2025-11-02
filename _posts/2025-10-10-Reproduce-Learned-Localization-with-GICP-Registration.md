---
layout: post
title: "ReLL: Reproduce Learned Localization with GICP Registration of Lidar&DSM"
date: 2025-10-10 16:00:00 +0000
categories: Localization
tags: [localization, map, ml, autonomous, DSM, lidar, imagery, geospatial]
---


## 1 Intro
This post introduce the reproduction of the ICRA Paper: Evaluating Global Geo-alignment for Precision Learned Autonomous Vehicle Localization using Aerial Data( https://arxiv.org/abs/2503.13896). This post will mainly talk about the detail the paper did not mentions, including the data preparing, data outline,  model design ,train progress. also mentions the chanllengings and solution I have faced. 

All data pipeline method , training code open source  : ![GitHub]https://github.com/rongweiji/Rell



### 1.1 Outcome
- Implemation the data pipeline including GICP registration, and model trianing progress, prove the GICP registration gain good result. 
- Experiment several loss funciotn contrscutrion , softmax, improved gaussian fit method. at the 0.2m resolution dataset gain rms 0.1m level translation acurracy.
- Explore the edge case mention in the original paper. Propose a factor relate with the lidar point fillment rate reason. 

### 1.2 Core of original paper
GICP: (https://www.roboticsproceedings.org/rss05/p21.pdf) is brought to be a method for resgistration, and the embeding encoder model training and vali by translation metircs. It approved the GICP included can improve high resulotion localization in the imagery coordinate. 

![Comapre GICP before and after ](/img/GICP%20align%20comapre.png)
Fig0 After applied the GICP wieh the lidar point registe to the DSM points we gain better alignemnt when we apply the projection to imagery which enhence the learned localization model training .


Learned Localization: Train encoder for V&M, sliding the embeding image to measure the similartiy within a search window. Define the cost volume to measure the cross-correlation,   and stack the shift to measure the shift capcity or vibrate range . using both the pixel level offset and subpixel offset by Gaussion fit to gain the presice peak.


### 1.3 My Data 
- Lidar source: argoverse-2 data https://www.argoverse.org/av2.html
- DSM : Bexar & Travis Counties Lidar | 2021 https://data.geographic.texas.gov/collection/?c=447db89a-58ee-4a1b-a61f-b918af2fb0bb
- Imagery :  Capital Area Council of Governments Imagery | 2022  :https://data.geographic.texas.gov/collection/?c=a15f67db-9535-464e-9058-f447325b6251  , resolution 0.3047 
- Coordinate :UTM , 
- Trainning sample amount: 1108




## 2 Gaussian fit method 


in my trainning progress i applied the softmax probabilities  method for loss function to Backpropagating the loss update the model . But the guassian fit only for the infer. 
On the mix dataset, i use the guassian fit to perform the refine position solve gain better result than softmax reuslt.

- Extract 3×3 patch centered on the discrete peak and clamp tiny values.
- Take log of pixel values → Gaussian amplitude becomes a quadratic surface.
- Use neighbor values to estimate first and second derivatives.
- Solve for stationary point: dx = −(d/dx) / (d2/dx2), dy = −(d/dy) / (d2/dy2).
- Clamp offsets to a safe range and add to integer center → sub‑pixel (x,y).
- Optionally sample score via bilinear interpolation at refined (x,y).



| **Method** | **RMS Errors (m)** | **P50 (Median) (m)** | **P99 (99th Percentile) (m)** | **Mean Distance (m)** | **Max Distance (m)** |
|:------------|:-------------------|:----------------------|:-------------------------------|:----------------------|:--------------------:|
| **Softmax Refinement** | **X:** 0.2273<br>**Y:** 0.2637<br>**Dist:** 0.3481 | **X:** 0.0711<br>**Y:** 0.1161<br>**Dist:** 0.1753 | **X:** 1.0376<br>**Y:** 1.0816<br>**Dist:** 1.2508 | 0.2498 | 1.6772 |
| **Gaussian Peak Refinement** | **X:** 0.1907<br>**Y:** 0.2222<br>**Dist:** 0.2928 | **X:** 0.0690<br>**Y:** 0.1154<br>**Dist:** 0.1701 | **X:** 0.9560<br>**Y:** 1.0050<br>**Dist:** 1.2029 | 0.2090 | 1.6772 |

Based on my data the guassian fit method gain better converge than softmax, but because of undiffenretiable guassian fit we can not applied it into the model training .

![Position Refinement Comparison](/img/Position%20Refinment%20Comparison.png)
Fig1. comapre 3 refined cross-corralation peak finding result at the mix dataset 

![Compare Peak Finding Method](/img/Comapre%20peak%20finding%20method.png)
Fig2. compare 3 refined peak finding method, most time gaussain fit method can gain equaivlant result as softmax 

![Compare Peak Finding Method 2](/img/Comparepeakfindingmethod2.png)
Fig3. sometime the guassian fit method gains better result

## 3 Fillment rate

In the original paper describe the situaiton is when throughing the underpass the localization result is not reliable, same sitatuation in my case. I find it related with the lidar points fillment, when passing through the bridge in this example fig4 we see it can not cover whole arear . so there is fewer feature when we trainning it


![Compare example filment rate](/img/fillment_comapre.png)
Fig4. Passing through bridge, the lidar pints cloud can not cover all area , compare with the better coverrage which gain better offset result and more convergence of the cross-correlation peak seeking.

Based on this situation , it explain why bigger size of the area datase whhich i have tried bigger area 0.3047reosluton with 100mx100m arear which result woarse metrics . 
Then the 0.2 resolution , smaller area aligneed with the paper gain better model training 

| Scale  | Resolution (m) | Raster Size | Range (m) | x_rms (m, UTM) | y_rms (m, UTM) | θ (deg) |
|--------|----------------|-------------|------------|----------------|----------------|----------|
| Scale 1 | 0.3047         | 329 × 329   | 100 × 100  | 0.483          | 0.423          | 0.7083   |
| Scale 2 | 0.2            | 150 × 150   | 30 × 30    | 0.105          | 0.106          | 0.3980   |


I also compare the relatition between offset distance with the fill rate 
![Compare 2 dataset of fill rate](/img/fillment%20rate%20statics.png)
Fig5. Bigger range of the data contains the lower fill rate, hard for represent the geomertry feature in model learning.


## 4 Challengings 

- Pre-procssing about the GICP, GICP perform good for registration different data which can be align with eachother .
