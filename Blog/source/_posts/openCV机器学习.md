---
title: openCV机器学习
date: 2017-12-15 20:37:26
tags: 机器学习
categories: 学习
mathjax: true
---
上周看代码的时候发现自己把机器学习的代码写了个开头，然后就烂尾了，想着既然开了个头，干脆就花一些时间写下去得了，反正是使用OpenCV去做一个封装，想必也没有多难，所以就开始写了。其实关于模式识别和机器学习的东西很早就开始做了，但是总是觉得做的并不太好，所以感觉有点遗憾，也没有能够留下一点什么东西。
总的来说以前主要做的东西是关于MNIST手写数字识别库的一些东西，手写数字识别库是一个用于机器学习的开源库，在机器学习中这个库使用得很广泛，其中包含了60000个训练样本和1000预测样本，这个库的格式很简单，首先是4字节的int型变量，表示样本的个数，然后是两个四字节的变量表示每个手写数字的大小，实际上训练样本是60000个，每个样本影像的大小为28*28，在读取数据头的时候注意，需要字节取反，所以整个读取存在一个字节取反的问题，字节取反的代码如下：
```c++
void mnistFile::swapBuffer(char* buf)
{
	char temp;
	temp = *(buf);
	*buf = *(buf+3);
	*(buf+3) = temp;

	temp = *(buf+1);
	*(buf+1) = *(buf+2);
	*(buf+2) = temp;
}
```
由于是通过OpenCV进行机器学习，所以我们最后数据都要转换为OpenCV的Mat格式，实际上在数据构造的时候可以直接讲数据转换为Mat就好了

```c++
void CVMachineLearningTrain::CV_GetMnistTrainData(const char* pathMnist, const char* pathLabel, Mat &trianMat, Mat &labelMat)
{
	ifstream ifs(pathMnist, ios_base::binary);
	char magicNum[4], ccount[4], crows[4], ccols[4];
	ifs.read(magicNum, sizeof(magicNum)); ifs.read(ccount, sizeof(ccount));
	ifs.read(crows, sizeof(crows)); ifs.read(ccols, sizeof(ccols));
	mnistFile tmp;
	int count, rows, cols;
	tmp.swapBuffer(ccount); tmp.swapBuffer(crows); tmp.swapBuffer(ccols);
	memcpy(&count, ccount, sizeof(count));
	memcpy(&rows, crows, sizeof(rows));
	memcpy(&cols, ccols, sizeof(cols));

	ifstream lab_ifs(pathLabel, ios_base::binary);
	lab_ifs.read(magicNum, sizeof(char) * 4);
	lab_ifs.read(ccount, sizeof(char) * 4);
	int intmagicNum, ccountint;

	tmp.swapBuffer(magicNum);
	tmp.swapBuffer(ccount);
	memcpy(&intmagicNum, magicNum, sizeof(magicNum));
	memcpy(&ccountint, ccount, sizeof(ccount));

	unsigned char* imgData = new unsigned char[rows*cols];
	float* imgDataf = new float[rows*cols];
	float* label = new float[10];
	int num = 0;
	double totalNormal = 0;
	Mat tmp1(count, rows*cols, CV_32FC1);
	Mat labelsMat(count, 10, CV_32FC1);
	char tmpLabel;

	for (int j = 0; j < count; ++j)
	{
		ifs.read((char*)imgData, rows * cols);
		for (int i = 0; i < rows * cols; ++i)
			imgDataf[i]= float(imgData[i])/255.f;

		memset(label, 0, sizeof(float) * 10);
		lab_ifs.read(&tmpLabel, sizeof(tmpLabel));
		int tlabel = tmpLabel;
		label[tlabel] = 1.0f;

		memcpy(tmp1.data + j*rows*cols*sizeof(float), imgDataf, rows*cols*sizeof(float));
		memcpy(labelsMat.data + j*10*sizeof(float), label, 10*sizeof(float));
	}

	trianMat = tmp1.clone();
	labelMat = labelsMat.clone();
	float* p = (float*)labelsMat.data;
	lab_ifs.close();
	ifs.close();

	delete[]imgData; imgData = NULL;
	delete[]label; label = NULL;
	delete[]imgDataf; imgDataf = NULL;
}
```
获取数据之后保存在Mat之中，从以上代码我们可以看到，实际上我们获取了28*28的数据，一共60000个，组成了一个60000x784的矩阵，而训练结果我们使用一个60000x10的矩阵表示，0-9数字，为哪个数字则一个10维的向量对应的一项为1其他项都为0，对于不同的训练数据类型应该采取不同的转换样式，如果为BP神经网络训练，则Label可以为10维的向量；若训练方式为SVM则结果为1维int类型的结果向量。
则使用OpenCV进行分类的代码为：
```c++
void CVMachineLearningTrain::CV_ANN_BP_Train(const char* pathDataset, const char* pathLabelSet, const char* pathNet, DatasetTypes datasetType)
{
	Ptr<ANN_MLP> ann = ANN_MLP::create();
	int layers[3] = { 28 * 28,15,10 };
	Mat_<int> layerSize(1, 3);
	memcpy(layerSize.data, layers, sizeof(int) * 3);
	ann->setLayerSizes(layerSize);
	ann->setActivationFunction(ANN_MLP::SIGMOID_SYM, 1, 1);
	ann->setTermCriteria(TermCriteria(TermCriteria::MAX_ITER + TermCriteria::EPS, 300, FLT_EPSILON));
	ann->setTrainMethod(ANN_MLP::BACKPROP, 0.001);
	Mat trainMat, labelMat;
	if (datasetType == DATASET_MNIST)
		CV_GetMnistTrainData(pathDataset, pathLabelSet, trainMat, labelMat);
	Ptr<TrainData> tData = TrainData::create(trainMat, ROW_SAMPLE, labelMat);
	printf("BP Netural Network train~ing...\n");
	ann->train(tData);
	printf("-done\n");
	ann->save(pathNet);
}

void CVMachineLearningTrain::CV_SVM_Train(const char* pathDataset,double C ,const char* pathLabelSet, const char* pathSVM, DatasetTypes datasetType)
{
	Mat trainData;
	Mat labelsMat;
	if(datasetType== DATASET_MNIST)
		CV_GetMnistTrainData(pathDataset, pathLabelSet, trainData, labelsMat);

	float* labelf =(float*)labelsMat.data;
	Mat labelsMatAdapter=Mat::zeros(labelsMat.rows, 1, CV_32SC1);
	for (int i = 0; i < labelsMat.rows; ++i)
	{
		int j = 0;
		for (j = 0; j < labelsMat.cols; ++j)
		{
			if (abs(labelf[i*labelsMat.cols +j] - 1) < 0.0001)
				break;
		}
		labelsMatAdapter.at<int>(i, 0)  = j;
	}

	Ptr<TrainData> tData = TrainData::create(trainData, ml::ROW_SAMPLE, labelsMatAdapter);
	Ptr<SVM> svm = SVM::create();
	svm->setType(SVM::C_SVC);
	svm->setKernel(SVM::POLY); //SVM::LINEAR;
	svm->setDegree(1);
	svm->setGamma(0.3);
	svm->setCoef0(1.0);
	svm->setNu(1);
	svm->setP(0.5);
	svm->setTermCriteria(TermCriteria(TermCriteria::MAX_ITER + TermCriteria::EPS, 10000, 0.01));
	svm->setC(C);
	printf("SVM train~ing...\n");
	svm->train(tData);
	printf("-done\n");
	svm->save(pathSVM);

	///////////////////////////////////////////////////////////
}

void CVMachineLearningTrain::CV_LogisticRegression_Train(const char* pathDataset, const char* pathLabelSet, const char* pathLogisticRegression, DatasetTypes datasetType)
{
	Mat trainData;
	Mat labelsMat;
	if (datasetType == DATASET_MNIST)
		CV_GetMnistTrainData(pathDataset, pathLabelSet, trainData, labelsMat);
	float* labelf = (float*)labelsMat.data;
	Mat labelsMatAdapter = Mat::zeros(labelsMat.rows, 1, CV_32FC1);
	for (int i = 0; i < labelsMat.rows; ++i)
	{
		int j = 0;
		for (j = 0; j < labelsMat.cols; ++j)
		{
			if (abs(labelf[i*labelsMat.cols + j] - 1) < 0.0001)
				break;
		}
		labelsMatAdapter.at<float>(i, 0) = j;
	}
	float* data = (float*)labelsMatAdapter.data;
	Ptr<LogisticRegression> lr1 = LogisticRegression::create();
	lr1->setLearningRate(0.001);
	lr1->setIterations(10);
	lr1->setRegularization(LogisticRegression::REG_L2);
	lr1->setTrainMethod(LogisticRegression::BATCH);
	lr1->setMiniBatchSize(1);
	printf("logistic regression train~ing...\n");
	lr1->train(trainData, ROW_SAMPLE, labelsMatAdapter);
	printf("-done\n");
	lr1->save(pathLogisticRegression);
}
```
训练得到训练好了的分类库输出到文件中，在进行判别的时候直接从文件中读取训练结果进行判别。
则进行判别的代码比较简单，在这里不进行详细描述，详细代码在Git上可以获取：https://github.com/wuweiFrank/rsProcess/tree/master/rsProcess/machineLearning 关于相关算法的具体描述，以后有时间会陆续进行记录和描述。
