---
title: tensorflow-三十一弹
date: 2020-03-22 10:55:25
tags: tensorflow学习
categories: 学习
mathjax: true
---
&nbsp;&nbsp;&nbsp;&nbsp;前一段时间熟悉了一下tensorflow2.0以及kersa，然后通过mnist数据集构建了一个简单的深度网络进行了学习，基于上一弹的基础我们来进行一些更加深入的学习，数据集同样是采用mnist数据集，我们来训练一个GAN网络，首先我们对GAN的原理进行分析：
## GAN原理分析
&nbsp;&nbsp;&nbsp;&nbsp;GAN, Generative Adversarial Networks又被称为生成对抗网络[<sup>1</sup>](#refer-anchor-1)，在这个网络模型中至少需要两个部分，分别成为生成器和识别器，其中生成器主要作用在于根据一个随机输入生成需要生成的对象，识别器可以i认为就是一个简单的神经网络，对生成器生成的数据进行识别。在此过程中需要明确的是代价函数，针对生成器和识别器我们都需要指定代价函数，具体的函数形式我们后续再讨论，首先我们先讨论一下生成器与识别器的模式。
### 生成器
&nbsp;&nbsp;&nbsp;&nbsp;实际上生成器可以简单的理解为一个分布函数，只是这个分布函数的参数是一个深度神经网络，根据一个输入以及目标label得到一个确定的输出，具体代码为：
```python
def acgan_generator():
    model = Sequential()

    model.add(layers.Dense(128 * 7 * 7, activation="relu", input_dim=100))
    model.add(layers.Reshape((7, 7, 128)))
    model.add(layers.BatchNormalization(momentum=0.8))
    model.add(layers.UpSampling2D())
    model.add(layers.Conv2D(128, kernel_size=3, padding="same"))
    model.add(layers.Activation("relu"))
    model.add(layers.BatchNormalization(momentum=0.8))
    model.add(layers.UpSampling2D())
    model.add(layers.Conv2D(64, kernel_size=3, padding="same"))
    model.add(layers.Activation("relu"))
    model.add(layers.BatchNormalization(momentum=0.8))
    model.add(layers.Conv2D(1, kernel_size=3, padding='same'))
    model.add(layers.Activation("tanh"))

    model.summary()

    noise = layers.Input(shape=(100,))
    label = layers.Input(shape=(1,), dtype='int32')
    label_embedding = layers.Flatten()(layers.Embedding(10, 100)(label))

    model_input = layers.multiply([noise, label_embedding])
    img = model(model_input)
    #将模型输出
    keras.utils.plot_model(model, to_file='./data/2_Generator.png', show_shapes=True)
    return tf.keras.Model([noise, label], img)
```
这个生成器的结构为：
![生成器](https://blogimage-1251632003.cos.ap-guangzhou.myqcloud.com/2_Generator.png)
为了简单起见我们结合代码与程序结构同时进行分析，首先我们的生成器是一个随机的输入向量，这个向量是一个100维的随机向量，同时输入一个label值，通过label值来标识需要模拟哪个数值（mnist只有10个值），这个100维向量通过神经网络转换维128*7*7的向量，步骤包括：
* 向量转换为二维->归一化->向上采样维128*（14*14）的向量->卷积操作->激活函数->归一化->向上采样维64*（28*28）的向量->卷积操作->激活函数->归一化为1&（27*28）的向量->卷积操作->激活函数

最后得到一个生成的28*28的手写数字以及输入的待识别的标识，函数最后将构建的模型输出，实际上生成器就完成了，简单的说就是根据一个随机的输入值构建了一个生成28*28大小的影像。

### 识别器
&nbsp;&nbsp;&nbsp;&nbsp;实际上识别器的主要作用在于识别图像是否由生成器生成，在我们这个目标中识别器的作用是识别通过生成器生成的手写数字是否为对应的数字，识别器的具体代码为：
```python
def acgan_discriminator():
    model = Sequential()
    model.add(layers.Conv2D(16, kernel_size=3, strides=2, input_shape=(28,28,1), padding="same"))
    model.add(layers.LeakyReLU(alpha=0.2))
    model.add(layers.Dropout(0.25))
    model.add(layers.Conv2D(32, kernel_size=3, strides=2, padding="same"))
    model.add(layers.ZeroPadding2D(padding=((0,1),(0,1))))
    model.add(layers.LeakyReLU(alpha=0.2))
    model.add(layers.Dropout(0.25))
    model.add(layers.BatchNormalization(momentum=0.8))
    model.add(layers.Conv2D(64, kernel_size=3, strides=2, padding="same"))
    model.add(layers.LeakyReLU(alpha=0.2))
    model.add(layers.Dropout(0.25))
    model.add(layers.BatchNormalization(momentum=0.8))
    model.add(layers.Conv2D(128, kernel_size=3, strides=1, padding="same"))
    model.add(layers.LeakyReLU(alpha=0.2))
    model.add(layers.Dropout(0.25))

    model.add(layers.Flatten())
    model.summary()

    img = layers.Input(shape=(28,28,1))

    # Extract feature representation
    features = model(img)

    # Determine validity and label of the image
    validity = layers.Dense(1, activation="sigmoid")(features)
    label = layers.Dense(10, activation="softmax")(features)
    
    keras.utils.plot_model(model, to_file='./data/2_Discriminator.png', show_shapes=True)
    return tf.keras.Model(img, [validity, label])
```
![识别器](https://blogimage-1251632003.cos.ap-guangzhou.myqcloud.com/2_Discriminator.png)
识别器可以简单的理解为是一个识别输入的图像是否为对应数组的一个深度网络，其输入是影像，输出是数字的10维向量，对于识别器的识别过程，在以前就多此提到过，在这里没有必要再多提了，通过识别器可以对生成器生成的结果进行识别给出判断结果对生成器进行优化，同时识别器不仅要识别数字结果还需要识别validity，这个标识的主要作用在于标识输入的参数是由生成器提供还是真实数据。

### 训练过程
```python
def train(epochs, batch_size=128, sample_interval=10):
    # Build and compile the discriminator
    optimizer = Adam(0.0002, 0.5)
    losses = ['binary_crossentropy', 'sparse_categorical_crossentropy']
    discriminator = acgan_discriminator()
    discriminator.compile(loss=losses,
        optimizer=optimizer,
        metrics=['accuracy'])

    # Build the generator
    
    generator = acgan_generator()

    # The generator takes noise and the target label as input
    # and generates the corresponding digit of that label
    noise = layers.Input(shape=(100,))
    label = layers.Input(shape=(1,))
    img = generator([noise, label])

    # For the combined model we will only train the generator
    discriminator.trainable = False

    # The discriminator takes generated image as input and determines validity
    # and the label of that image
    valid, target_label = discriminator(img)

    # The combined model  (stacked generator and discriminator)
    # Trains the generator to fool the discriminator
    combined = tf.keras.Model([noise, label], [valid, target_label])
    combined.compile(loss=losses,optimizer=optimizer)
    keras.utils.plot_model(combined, to_file='./data/2_Combined.png', show_shapes=True)

    # Load the dataset
    (X_train, y_train), (_, _) = load_mnist_data()

    # Configure inputs
    X_train = (X_train.astype(np.float32) - 127.5) / 127.5
    X_train = np.expand_dims(X_train, axis=3)
    y_train = y_train.reshape(-1, 1)

    # Adversarial ground truths
    valid = np.ones((batch_size, 1))
    fake = np.zeros((batch_size, 1))

    for epoch in range(epochs):
        # ---------------------
        #  Train Discriminator
        # ---------------------

        # Select a random batch of images
        idx = np.random.randint(0, X_train.shape[0], batch_size)
        imgs = X_train[idx]
        # Image labels. 0-9 
        img_labels = y_train[idx]

        # Sample noise as generator input
        noise = np.random.normal(0, 1, (batch_size, 100))
        # The labels of the digits that the generator tries to create an
        # image representation of
        sampled_labels = np.random.randint(0, 10, (batch_size, 1))
        # Generate a half batch of new images
        gen_imgs = generator.predict([noise, sampled_labels])

        # Train the discriminator
        d_loss_real = discriminator.train_on_batch(imgs, [valid, img_labels])
        d_loss_fake = discriminator.train_on_batch(gen_imgs, [fake, sampled_labels])
        d_loss = 0.5 * np.add(d_loss_real, d_loss_fake)

        # ---------------------
        #  Train Generator
        # ---------------------

        # Train the generator
        g_loss = combined.train_on_batch([noise, sampled_labels], [valid, sampled_labels])

        # Plot the progress
        print ("%d [D loss: %f, acc.: %.2f%%, op_acc: %.2f%%] [G loss: %f]" % (epoch, d_loss[0], 100*d_loss[3], 100*d_loss[4], g_loss[0]))

        # If at save interval => save generated image samples
        if epoch % sample_interval == 0:
            #save_model(generator,discriminator)
            sample_images(epoch,generator)
    save_model(generator,discriminator)
```
代码也不是很难，主要过程在于训练的过程，在训练的过程中我们通过真实数据与fake数据的交替训练对生成器和识别器进行训练，识别器需要识别的结
```python
d_loss_real = discriminator.train_on_batch(imgs, [valid, img_labels])
d_loss_fake = discriminator.train_on_batch(gen_imgs, [fake, sampled_labels])
d_loss = 0.5 * np.add(d_loss_real, d_loss_fake)
```
以上代码就是识别器识别的过程，实际上我们识别器需要保证识别的label和vaild同时准确才行，所以识别的损失函数为识别真实数据的损失函数与生成器生成数据的损失函数；在完成识别器训练后就需要对生成器进行训练，对生成器的训练过程实际上损失函数就是输入的noise以及需要识别的数字，然后通过识别器能够识别出是是否是生成器生成的数据以及是否为给定的数值，据此对生成器进行训练，通过交替训练识别器与生成器达到最后生成器能够进行最好的识别的效果。
### 结果与讨论
&nbsp;&nbsp;&nbsp;&nbsp;通过以上过程可以得到一个稳定的生成器与识别器，实际上GAN最大的作用还是在于训练生成器，使得生成器生成的数据能够最佳拟合真实数据，达到模拟仿真的目的。同时GAN也被广泛的应用于图像修复，图像生成，音乐生成，围棋[<sup>2</sup>]("refer-anchor-2")[<sup>3</sup>]("refer-anchor-3")[<sup>4</sup>]("refer-anchor-4")等领域，实际上GAN的出现使得计算器自学习能力向前迈进了一大步，下面展示通过GAN训练的手写数字模拟的效果：
![GAN模拟手写数据结果](https://blogimage-1251632003.cos.ap-guangzhou.myqcloud.com/2_GANResult.png)
上图为模拟的结果，模拟的数字为1，文件名为迭代的次数，每次迭代的batch大小为100，可以看出迭代次数小的时候模拟结果比较差，当迭代次数增加，生成器效果慢慢编号，最后随着迭代次数的增加，生成器生成的数据已经能够很好的模拟手写数字的真实数据了。

## 参考
<div id="refer-anchor-1"></div>
- [1] Goodfellow, I. J., Pouget-Abadie, J., Mirza, M., Xu, B., Warde-Farley, D.,
Ozair, S., Courville, A., and Bengio, Y. (2014b). Generative adversarial networks. In NIPS’2014 .
<div id="refer-anchor-2"></div>
- [2] Yijun Li, Sifei Liu, Jimei Yang,et.al. Generative Face Completion[C]// IEEE Conference on Computer Vision & Pattern Recognition. IEEE, 2017.
<div id="refer-anchor-3"></div>
- [3] Jean-Marc Valin, Jan Skoglund. LPCNET: Improving Neural Speech Synthesis through Linear Prediction[C]// Icassp IEEE International Conference on Acoustics. IEEE, 2019.
<div id="refer-anchor-4"></div>
- [4] Jim X. Chen. The Evolution of Computing: AlphaGo[J]. Computing in Science & Engineering, 2016, 18(4):4-7.