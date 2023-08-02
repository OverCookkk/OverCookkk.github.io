dreambooth：解决泛化性差的问题

lora：减少模型的训练参数







### dreambooth

工作方式实际上是通过改变模型本身的结构，它主要有两个输入，文字和图片，假如，你需要让模型识别一只柯基，你需要输入多张柯基的图片，同时，还需要输入文本（一个标识符），教会模型使用这个标识符与柯基犬的概念联系起来。

原理：文本会转换成文本embedding，也就是向量，向量包含了语义信息。

。。。。。









### Textual Inversion



与dreambooth原理大体相同，不同的是，产生的loss，执行梯度更新时不是直接惩罚diffusion model，而是更新了文本的向量，使得文本向量越来越接近这个柯基，也就是文本能完美的向模型描述了这只柯基。

Textual Inversion不需要更新diffusion model，只需要



在文本的向量空间中找到新的伪装词，通过这个伪装词去捕获高级语义和精细的视觉细节，在大模型生成图片的时候，把这个反转文本嵌入到大模型中，让大模型学习到了这个伪装词的新概念。





### Lora

![lora原理图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/lora%E5%8E%9F%E7%90%86%E5%9B%BE.png)

![lora层示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/lora%E5%B1%82%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

lora在维持原来的参数不变的基础下，插入一个新的计算层，只需要调整新的训练层参数就可以了，随着训练的进行，去更新这些新加入的计算层；论文中介绍训练参数数量降低10000倍，GPU内存需求降低3倍，训练速度快，模型小（几十兆到上百兆），即插即用。





### Hypernetworks

![Hypernetworks原理图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/Hypernetworks%E5%8E%9F%E7%90%86%E5%9B%BE.png)

它的工作原理基本上与lora相同，它在基础模型中也是插入了一些新的计算层，不同的是，损失loss更新不是直接这些中间层，而是更新一个学习如何创建这些中间层的Hypernetworks网络（超网络），最后Hypernetworks网络会输出这些中间层（矩阵）插入到基础模型中。

