---
title: tensorflow-二十二弹
date: 2018-01-21 11:36:10
tags: tensorflow学习
categories: 学习
---
上次我们谈到了Q-Learning算法，实际上就是构建一个Q表的过程，那么我们怎么将深度网络和Q-Learning算法相结合，这个就是我们这篇文章会提到的问题．我们依旧拿上次的那个FlappyBird的例子来进行说明，在这个例子中如果我们通过Q-Learning算法进行处理，那么实际上Q表就是整个影像范围的所有像素，而每一个像素又存着0-255的像素值的情况，即使没有0-255的像素值这么多，整个Q表也是一个极大的值，在这样的情况下求出Q表中的每一个值明显不现实，需要减小Q表的规模，在这样的情况下我们就引入了深度学习算法．  
在上一篇中我们提到过实际上我们也不一定需要一个Q表，我们只需要拟合一个Q值函数就可以了，也就是能够估计Q表的分布，就能够进行计算了，简单的来说我们的模型为：  
<img src="http://blogimage-1251632003.cosgz.myqcloud.com/DQN-%E7%8A%B6%E6%80%81%E8%AF%B4%E6%98%8E.png" />  
在传统的Q-Learning算法中，其中Q表为一个表，而对于某些任务，由于Q表过大，不宜用一个表来进行展示，因此我们将Q表转换为一个能够根据输入解算出对应所有action的reward的所有值的一个网络，因此我们的整个算法过程就分为两个部分：１.求解Q网络的过程;2.根据Q网络进行判断的过程；好了，废话也不多说了，我们直接解析整个代码：

```python
#!/usr/bin/env python
from __future__ import print_function

import tensorflow as tf
import cv2
import sys
sys.path.append("game/")
import wrapped_flappy_bird as game
import random
import numpy as np
from collections import deque

GAME = 'bird' # the name of the game being played for log files
ACTIONS = 2 # number of valid actions
GAMMA = 0.99 # decay rate of past observations
OBSERVE = 100000. # timesteps to observe before training
EXPLORE = 2000000. # frames over which to anneal epsilon
FINAL_EPSILON = 0.0001 # final value of epsilon
INITIAL_EPSILON = 0.0001 # starting value of epsilon
REPLAY_MEMORY = 50000 # number of previous transitions to remember
BATCH = 32 # size of minibatch
FRAME_PER_ACTION = 1

def weight_variable(shape):
    initial = tf.truncated_normal(shape, stddev = 0.01)
    return tf.Variable(initial)

def bias_variable(shape):
    initial = tf.constant(0.01, shape = shape)
    return tf.Variable(initial)

def conv2d(x, W, stride):
    return tf.nn.conv2d(x, W, strides = [1, stride, stride, 1], padding = "SAME")

def max_pool_2x2(x):
    return tf.nn.max_pool(x, ksize = [1, 2, 2, 1], strides = [1, 2, 2, 1], padding = "SAME")

def createNetwork():
    # network weights
    W_conv1 = weight_variable([8, 8, 4, 32])
    b_conv1 = bias_variable([32])

    W_conv2 = weight_variable([4, 4, 32, 64])
    b_conv2 = bias_variable([64])

    W_conv3 = weight_variable([3, 3, 64, 64])
    b_conv3 = bias_variable([64])

    W_fc1 = weight_variable([1600, 512])
    b_fc1 = bias_variable([512])

    W_fc2 = weight_variable([512, ACTIONS])
    b_fc2 = bias_variable([ACTIONS])

    # input layer
    s = tf.placeholder("float", [None, 80, 80, 4])

    # hidden layers
    h_conv1 = tf.nn.relu(conv2d(s, W_conv1, 4) + b_conv1)
    h_pool1 = max_pool_2x2(h_conv1)

    h_conv2 = tf.nn.relu(conv2d(h_pool1, W_conv2, 2) + b_conv2)
    #h_pool2 = max_pool_2x2(h_conv2)

    h_conv3 = tf.nn.relu(conv2d(h_conv2, W_conv3, 1) + b_conv3)
    #h_pool3 = max_pool_2x2(h_conv3)

    #h_pool3_flat = tf.reshape(h_pool3, [-1, 256])
    h_conv3_flat = tf.reshape(h_conv3, [-1, 1600])

    h_fc1 = tf.nn.relu(tf.matmul(h_conv3_flat, W_fc1) + b_fc1)

    # readout layer
    readout = tf.matmul(h_fc1, W_fc2) + b_fc2

    return s, readout, h_fc1

def trainNetwork(s, readout, h_fc1, sess):
    # define the cost function
    a = tf.placeholder("float", [None, ACTIONS])
    y = tf.placeholder("float", [None])
    readout_action = tf.reduce_sum(tf.multiply(readout, a), reduction_indices=1)
    cost = tf.reduce_mean(tf.square(y - readout_action))
    train_step = tf.train.AdamOptimizer(1e-6).minimize(cost)

    # open up a game state to communicate with emulator
    game_state = game.GameState()

    # store the previous observations in replay memory
    D = deque()

    # printing
    a_file = open("logs_" + GAME + "/readout.txt", 'w')
    h_file = open("logs_" + GAME + "/hidden.txt", 'w')

    # get the first state by doing nothing and preprocess the image to 80x80x4
    do_nothing = np.zeros(ACTIONS)
    do_nothing[0] = 1
    x_t, r_0, terminal = game_state.frame_step(do_nothing)
    x_t = cv2.cvtColor(cv2.resize(x_t, (80, 80)), cv2.COLOR_BGR2GRAY)
    ret, x_t = cv2.threshold(x_t,1,255,cv2.THRESH_BINARY)
    s_t = np.stack((x_t, x_t, x_t, x_t), axis=2)

    # saving and loading networks
    saver = tf.train.Saver()
    sess.run(tf.initialize_all_variables())
    checkpoint = tf.train.get_checkpoint_state("saved_networks")
    if checkpoint and checkpoint.model_checkpoint_path:
        saver.restore(sess, checkpoint.model_checkpoint_path)
        print("Successfully loaded:", checkpoint.model_checkpoint_path)
    else:
        print("Could not find old network weights")

    # start training
    epsilon = INITIAL_EPSILON
    t = 0
    while "flappy bird" != "angry bird":
        # choose an action epsilon greedily
        readout_t = readout.eval(feed_dict={s : [s_t]})[0]
        a_t = np.zeros([ACTIONS])
        action_index = 0
        if t % FRAME_PER_ACTION == 0:
            if random.random() <= epsilon:
                print("----------Random Action----------")
                action_index = random.randrange(ACTIONS)
                a_t[random.randrange(ACTIONS)] = 1
            else:
                action_index = np.argmax(readout_t)
                a_t[action_index] = 1
        else:
            a_t[0] = 1 # do nothing

        # scale down epsilon
        if epsilon > FINAL_EPSILON and t > OBSERVE:
            epsilon -= (INITIAL_EPSILON - FINAL_EPSILON) / EXPLORE

        # run the selected action and observe next state and reward
        x_t1_colored, r_t, terminal = game_state.frame_step(a_t)
        x_t1 = cv2.cvtColor(cv2.resize(x_t1_colored, (80, 80)), cv2.COLOR_BGR2GRAY)
        ret, x_t1 = cv2.threshold(x_t1, 1, 255, cv2.THRESH_BINARY)
        x_t1 = np.reshape(x_t1, (80, 80, 1))
        #s_t1 = np.append(x_t1, s_t[:,:,1:], axis = 2)
        s_t1 = np.append(x_t1, s_t[:, :, :3], axis=2)

        # store the transition in D
        D.append((s_t, a_t, r_t, s_t1, terminal))
        if len(D) > REPLAY_MEMORY:
            D.popleft()

        # only train if done observing
        if t > OBSERVE:
            # sample a minibatch to train on
            minibatch = random.sample(D, BATCH)

            # get the batch variables
            s_j_batch = [d[0] for d in minibatch]
            a_batch = [d[1] for d in minibatch]
            r_batch = [d[2] for d in minibatch]
            s_j1_batch = [d[3] for d in minibatch]

            y_batch = []
            readout_j1_batch = readout.eval(feed_dict = {s : s_j1_batch})
            for i in range(0, len(minibatch)):
                terminal = minibatch[i][4]
                # if terminal, only equals reward
                if terminal:
                    y_batch.append(r_batch[i])
                else:
                    y_batch.append(r_batch[i] + GAMMA * np.max(readout_j1_batch[i]))

            # perform gradient step
            train_step.run(feed_dict = {
                y : y_batch,
                a : a_batch,
                s : s_j_batch}
            )

        # update the old values
        s_t = s_t1
        t += 1

        # save progress every 10000 iterations
        if t % 10000 == 0:
            saver.save(sess, 'saved_networks/' + GAME + '-dqn', global_step = t)

        # print info
        state = ""
        if t <= OBSERVE:
            state = "observe"
        elif t > OBSERVE and t <= OBSERVE + EXPLORE:
            state = "explore"
        else:
            state = "train"

        print("TIMESTEP", t, "/ STATE", state, \
            "/ EPSILON", epsilon, "/ ACTION", action_index, "/ REWARD", r_t, \
            "/ Q_MAX %e" % np.max(readout_t))
        # write info to files
        '''
        if t % 10000 <= 100:
            a_file.write(",".join([str(x) for x in readout_t]) + '\n')
            h_file.write(",".join([str(x) for x in h_fc1.eval(feed_dict={s:[s_t]})[0]]) + '\n')
            cv2.imwrite("logs_tetris/frame" + str(t) + ".png", x_t1)
        '''

def playGame():
    sess = tf.InteractiveSession()
    s, readout, h_fc1 = createNetwork()
    trainNetwork(s, readout, h_fc1, sess)

def main():
    playGame()

if __name__ == "__main__":
    main()

```
好了,我们下面来分析以上代码，首先是头文件的引入和模块的定义，这个对于我们来说都没有特别的，在这里就不进行详细的介绍了，实际上就说明一下那个Action，由于在游戏中我们只有点击屏幕和不点击两个动作，因此Action也只有两个，我们先整体了解代码：
```python
  def weight_variable(shape):
  def bias_variable(shape):
  def conv2d(x, W, stride):
  def max_pool_2x2(x):
  def createNetwork():

  #训练网络
  def trainNetwork(s, readout, h_fc1, sess):
```
上面这几个过程都是深度卷积网络要干的事情，包括定义权重，定义偏置，定义卷积池化，以及创建网络的过程，这个过程在介绍深度卷积神经网络的时候就介绍过，在这里就不进行更加详细的介绍了，我们重点介绍的是这个训练的过程，实际上这个训练的过程也就是不停的对游戏进行play的过程，首先是初始化输入和输出，在tensorflow中使用palceholder来对变量进行初始化，然后定义行为，实际就是将行为输入我们的网络，然后得到下一个行为，而将我们通过网络得到的行为和我们定义的下一个行为的差值作为代价函数，通过这种方式进行训练；首先我们获取游戏的状态，game_state,然后将游戏的状态保存在一个队列中，定义两个动作，中间的过程就是一些预处理的过程，包括什么游戏初始化，初始化过程中采用[0,1]这个状态进行初始化，然后是一些预处理和读取checkpoint的过程，因为在训练过程中防止无法一次得到最佳结果，通过记录中间过程的方式可以断点重新计算；  
上面介绍了整个训练的准备过程，下面的重点就是介绍整个训练过程，整个训练过程从代码中我们看出，再每一论游戏的过程中，有两个选择策略，贪婪选择和随机选择，实际上这里存在一个折扣未来奖励的过程，因为我们的选取的当前最好的结果不一定是未来最好的结果，因此这里又一个epsilon的比例，在我们选择好了折扣未来奖励后我们有一定的机率选择随机或者当前最大值，更新epsilon，并根据我们的选取动作执行我们的行为，这样我们得到了下一个状态和当前状态的reward值，将状态存储在队列中，在队列中随机取状态，得到ｙ值，然后将得到的ｙ值和通过Q网络计算的y值进行比较对网络进行调整，每经过一定的步长之后对Q网络的参数进行更新，这样就完成了一次训练过程，然后不停的进行游戏，训练得到最佳结果；
