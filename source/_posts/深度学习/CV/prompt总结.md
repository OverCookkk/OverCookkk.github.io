# 风格





# 取景

**景别＋角度（拍摄方向+拍摄高度）＝取景**，控制镜头画面，包括景别和角度，景别是只被摄物体的大小，角度包括水平角度与垂直角度



## 控制“景别”的提示词

景别是指**由于在焦距一定时，摄影机与被摄体的距离不同，而造成被摄体在摄影机录像器中所呈现出的范围大小的区别**。

![景别位置图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E6%99%AF%E5%88%AB%E4%BD%8D%E7%BD%AE%E5%9B%BE.jpg)

- `extreme close-up` 特写镜头
- `close-up` 近景
- `medium close-up` 中近景
- `medium shot` 中景
- `long shot` 全景
- `medium full shot` 中全景
- `full shot` 全景

### **除了常规的景别外还有一些特殊作用的镜头：**

- `establishing shot` 定场镜头（视频开头，用来交代地点的镜头，通常是视野宽阔的远景）
- `point-of-view` 主观视角
- `cowboy shot` 西部牛仔镜头，见到上半身以及大腿（为了见到拔枪）
- `upper body` 上半身
- `full body` 全身

需要注意的是再加入景别提示词后，尽量不要加入面部描述 e.g. `beautiful face`这些脸部描述的提示词，否则多数都会变成半身照。

![景别效果图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E6%99%AF%E5%88%AB%E6%95%88%E6%9E%9C%E5%9B%BE.jpg)

出来的结果意外地有些 prompts 很相似，不过再配合其他 prompt 可以更稳定地控制距离。

- `extreme close-up`，`close-up` 跟 `medium close-up` 都是放大眼睛/面部为主，但有时`extreme close-up`会放大更多。
- `medium shot` ， `long shot` ， `medium full shot` 跟 `full shot` 看起来差不多，`medium shot`有时候会比 `full shot` 更近一点，都是臀部以上到头顶的位置，因为场景的问题几个 prompt 的距离可能会有些变化。
- `establishing shot` 的背景会比较明显，如果主体是建筑时人物可能会更小。
- `point-of-view` 角度会因应人物有点变化，背景通常比较开阔，角度跟主体未必是同一水平视角。

经测试后由近至远可用的镜头 - `extreme close-up` > `close-up` > `medium close-up` > `upper body` > `medium shot` > `medium full shot` > `full body` 。

而 `point-of-view` 跟 `establishing shot` 受环境有所影响，所以不适合控制距离。



## 拍摄角度=拍摄方向&拍摄高度

拍摄方向是水平方向的角度

拍摄高度是垂直方向的角度

### 控制“拍摄方向”的提示词

拍摄方向是指以被摄对象为中心，在同一水平面上围绕被摄对象四周选择摄影点。

掌握距离然后就是角度，由最基本的前后左右再加一些摄影角度，一样加上 `1.5` 权重，因为角度比较多我分成两类。

**多种视角 prompts**

- `front view` 正面
- `bilaterally symmetrical` 左右对称
- `side view` 侧面
- `back view` 后面
- `from above` 从上拍摄
- `from below` 从下拍摄
- `from behind` 后拍

其他特殊镜头

- `wide angle view` 广角镜头
- `fisheyes view` 鱼眼镜头
- `macro view` 微距

![拍摄方向生成图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E6%8B%8D%E6%91%84%E6%96%B9%E5%90%91%E7%94%9F%E6%88%90%E5%9B%BE.jpg)





### 控制“镜头高度”的提示词

![镜头高度示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E9%95%9C%E5%A4%B4%E9%AB%98%E5%BA%A6%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)



- `overhead shot` 俯视
- `top down` 由上向下
- `bird's eye view` 鸟瞰
- `high angle` 高角度
- `slightly above` 微高角度
- `straight on` 水平拍摄
- `hero view` 英雄视角
- `low view` 低视角
- `worm's eye view` 仰视
- `selfie` 自拍

![镜头高度生成图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/%E9%95%9C%E5%A4%B4%E9%AB%98%E5%BA%A6%E7%94%9F%E6%88%90%E5%9B%BE.jpg)

出来的结果跟字面表示的角度差不多，也有些角度其实是重复的，也有一些受字面影响而受到污染。

- `front view` `straight on` 就是正面，但不一定是绝对正面，`straight on` 因为水平拍摄的角度所以背景也不会歪。
- `bilateral symmetry` 正面兼左右对称，比正面更准确。
- `side view` 向左/向右都是随机的。
- `back view` 跟 `from behind` 都是背面， `back view` 会近一点，而且通常露背。
- `from above` `overhead shot` `high angle` `slightly above` 都是由高角度拍向主体， `overhead shot` 角度较高， `high angle` 会背景比较阔一些。
- `from below` 由下方偷拍的视角，天空通常会筒状变形。
- `wide angle` 背景会有一些桶状的变形 `fisheyes view` 的畸变效果会更强，但 `fisheyes view` 受到污染，总会拿著相机。
- `macro view` 变了拍花或微细的物件。
- `bird's eye view` 从高角度影高去同时会见到广阔的背景，但会有小鸟出现。
- `top down`  变成正上方被女生抱住的视角。
- `hero view` 角度不对，人物也受污染穿上了英雄战衣。
- `low view` 角度不算很低，有点怀疑没有效果。
- `worm's eye view` 完全错了，有很多虫及怪眼，跟角度完全没关係。
- `selfie` 人物会伸手自拍而且不会太远。

其中 `fisheyes view` 虽然会污染但因为视角比较特别还是有用的，但 `hero view` 跟 `worm's eye view` 及  `macro view` 受污染角度又不明显可以放弃。





# 光



## 光源

- `bloom` 令原本的光源更亮了，看头顶及肩膀有变亮的效果
- `soft lighting` 比较柔和的光源，面部有光，背部也有一点
- `hard lighting` 直接打在人物上的光，见到面部轮廓会比较突出
- `backlight` 就是背光，样子明显变暗了，肩膀及头发有背后来的光源
- `god rays` 另一种背光，由较高的位置射灯式射下来，见到头顶部份特别亮
- `volumetic lighting` 就像柔光版的背光，整体较暗，有点生化危机的感觉
- `sun light` 比较自然的阳光，连背景的树都会见到阳光照射
- `studio light` 左右都有光源打在面上，立体感很强，就像广告照
- `bioluminescent light` 本体在发光，就像萤火虫一样的夜光



## 影子和光追

- `detailed shadows` 就是加多点精细的影子，鼻子及衣服也有些光影出现
- `intricate tree shadow` 因为是树林所以就加点错综复杂的树影，看起来更真实
- `raytracing` 见到下巴位置有些衣服反射的光线