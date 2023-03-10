## Generative AI Stable Diffusion Workshop 手册

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image0.png" alt="太空歌剧院" style="zoom:150%;" />

本手册将介绍如何部署 All in one ai - stable diffusion WebUI。

### 1 准备部署环境

  **区域说明：**

  本实验涉及到的资源均放在美东1区(us-east-1)，请确保您在实验中使用此区域，并保持不变。

  **可用资源数量：**

  - VPC: 本实验需要创建 1 个 VPC，因此，您需要确认当前区域 VPC 的额度足够（默认上限为 5 / Region）。
  - EIP: 本实验需要创建 2 个 EIP，因此，您需要确认当前区域 EIP 的额度足够（默认上限为 5 / Region）。

  **Tips：**

  考虑到跨境网络的稳定性可能导致 SSH 中断，建议大家在 EC2 上使用 `screen` 或者 `tmux` 执行相关命令。

打开 AWS 控制台，在EC2 AMI 下找到分享的镜像（默认出现在 Private images 下面），镜像名称是 **GAI-SD-workshop-20230308** (美东一区 ami-0ae1f91257cdd5d0a，美西二区 ami-04bf95e4c34baa342)， 选中，然后基于此镜像创建一台 EC2 实例。该 EC2 实例类型选用 **m5.large** 即可（具备一定的网络带宽能力），EBS卷建议改为 gp3类型，要求能够 SSH 到该实例。

注意需要给该 EC2 附加一个 **IAM role**，该role要求具备如下权限：

- ECR 创建，Push，Pull
- S3 bucket 创建，上传，下载

简单起见，可以直接 **AmazonS3FullAccess** \ **AmazonEC2ContainerRegistryFullAccess** \ **SecretsManagerReadWrite** 这3个 IAM 策略。

### 2 部署方案

接下来我们开始部署该方案。

#### 2.1 SSH 到如上创建的 EC2 实例

```shell
ssh -i "YOUR_KEY.pem" ubuntu@YOUR_INSTANCE_FQDN.amazonaws.com
```

可以看到当前的目录是 ```/home/ubuntu```，以及有如下目录。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image1.png" style="zoom:100%;" />

执行 `screen` 命令以打开独立窗口:

```shell
screen -R aigc-demo
```

初始化环境变量：

```shell
# will create bucket with this name
export YOUR_BUCKET_NAME_FOR_WORKSHOP=gai-sd-workshop-`date +'%N'`

# default region
export REGION=us-east-1

# fetch your account id
export YOUR_ACCOUNT_ID=`aws sts get-caller-identity --query 'Account' | tr -d '"'`
```

#### 2.2 创建一个新的 S3 桶

建议创建一个新的 S3 桶，并记住桶名，后续使用。

```shell
aws --region $REGION s3 mb s3://$YOUR_BUCKET_NAME_FOR_WORKSHOP
```

#### 2.3 初始化 ECR 

接下来我们开始执行部署脚本，该脚本将创建相关的 ECR 镜像库，并推送镜像（为加快部署，我们已经在该 EC2 AMI中内置了相关容器镜像）。

进入 **deployment** 目录：

```shell
cd all-in-one-ai/deployment
```

切换到 **root** 用户，执行 **build_and_deploy.sh**脚本，填入前一步创建的 S3桶以及所在区域，如本次 workshop 我们采用美东1区，最后一个参数 **stable-diffusion-webui** 保持不变

```shell
sudo -E su
./build_and_deploy.sh s3://$YOUR_BUCKET_NAME_FOR_WORKSHOP $REGION stable-diffusion-webui
```

大约需要 30分钟，脚本执行完成，此时在 ECR将看到 4 个registry，分别是 **all-in-one-ai-stable-diffusion-webui-inference**，**all-in-one-ai-stable-diffusion-webui**，**all-in-one-ai-stable-diffusion-webui-training**，**all-in-one-ai-web**。



#### 2.4 通过 CloudFormation 部署 Web 前端

在上面创建的 S3 桶中，可以看到有 3个目录，分别为 **algorithms**，**codes**，**templates**。点击 templates并进入，将出现 15 个 yaml文件，找到并复制 `all-in-one-ai-webui-main.yaml` 的  object URL，或者使用如下命令获取URL。

```shell
aws s3 presign s3://$YOUR_BUCKET_NAME_FOR_WORKSHOP/templates/all-in-one-ai-webui-main.yaml | cut -d '?' -f 1
```

![image-20230221153757811](https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image2.png)



a. 进入 **CloudFormation** 控制台，新建一个 **Stack**，在 template 处选择 Amazon S3 URL，并填入上一步复制的 URL，然后下一步。

b. 填入 **stack name**，选择至少两个 AZ，在 **S3Bucket**处填入前面 S3 桶的名字（无需 s3://），其他部分使用默认值即可，下一步，直至开始创建。

此步骤大概20分钟完成，在 **Outputs** 可以看到两个输出， **WebPortalUrl** 和 **WebUIPortalUrl**。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image3.png" style="zoom:50%;" />



#### 2.5 准备预训练的模型

回到前面创建的 EC2，我们已经将 **v1-5-pruned.ckpt**，**v2-1_768-ema-pruned.ckpt** 两个模型已经放到了该 EC2，可以直接使用，无需自己再去下载。接下来我们把相关的模型文件上传到您账号下的SageMaker缺省S3 桶，如果对应桶不存在下列命令会自动创建.

```shell
aws --region $REGION s3 mb s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID
cd ~ubuntu
sh prepare.sh $REGION
```

该部分将上传 assets/model.tar.gz 以及 models/上面2个 models  到 S3。



### 3 使用 stable diffusion webui

本节我们将创建一个模型推理节点，并通过文字生成图片。

#### 3.1 创建模型推理节点

先打开 **WebUIPortalUrl**，该地址是 stable diffusion webui 的 portal，后面我们将基于此进行模型调试。
<img src="https://docs.ai.examples.pro/assets/images/stable_diffusion_webui_login.png" />

上面的用户名和密码使用下列方式得到，主要用于admin安全控制。里面的your-secret-name，形如 Administratorlogin-Hm8mbooqTyIv，可在上面的CloudFormation output找到，或在Secret Manager控制台找到。

```shell
cd ~ubuntu/all-in-one-ai/sagemaker/stable-diffusion-webui/tools

./secretmanager.sh [your-secret-name，形如 Administratorlogin-Hm8mbooqTyIv] us-east-1 get
```

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image5.png" style="zoom:50%;" />


在**用户** tab页下，注册个账户。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image10.png" style="zoom:50%;" />

然后打开上一步 CloudFormation 输出的 **WebPortalUrl**，该地址是 All in one ai 的 portal。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image4.png" style="zoom:50%;" />

在左边栏，行业模型下的概览，可以看到一个创建好的 名为 stable-diffusion-webui的对象，点击**开始体验**。如果没有出现下图这个模型，可以按照如下操作创建一个即可。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image6.png" style="zoom:50%;" />

---

**（分隔符开始）**

##### 如果没有出现上图模型，按如下步骤创建：

选择**创建您自己的行业模型**，在新出现界面中按下图**红框标识**的填写。

- **行业模型名称**和**行业模型描述**可以自定义；
- **行业模型算法**选择 Stable Diffusion Web UI；
- **必须**选择一个图标，图标文件大小不超过1MB；
- 算法特定信息处只输入 ```{}``` 即可；
- **模型示例数据 S3 URI** 留空；

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image21.png" alt="image21" style="zoom:50%;" />

完成后，点击**开始体验**，进入下面步骤。

**（分隔符结束）**

---



在新页面下点击**部署**。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image7.png" alt="image7" style="zoom:50%;" />

在新的页面下，填入**模型数据位置**，实例类型可以选择 **ml.g4dn.2xlarge**，然后提交。其中模型数据位置是之前在 EC2 中通过脚本上传到 S3桶中的 model.tar.gz 的 S3 URI，可以从 S3桶中找到（见下图），也可以使用下述命令获得：

```shell
echo s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/assets/model.tar.gz
```

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image9.png" alt="image9" style="zoom:40%;" />

#### 

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image8.png" alt="image8" style="zoom:40%;" />

等待模型部署成功，大概需要15分钟（本次实验的模型文件打包了多个模型，下载时间较长）。可以在SageMaker 控制台下的 Inference/Endpoint下面查看状态，等endpoint状态为 InService时，即可使用。

切换到 WebUI 界面，在**设置** tab 下，刷新 SageMaker Endpoint 和模型，出现刚刚部署出来的 endpoint，以及该 endpoint 上面的模型，我们可以选择 **v1-5-pruned.ckpt** 为例。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image11.png" style="zoom:40%;" />

#### 3.2 文字生成图片

在**文生图**页面，输入prompt 词，调整 steps 为50 （数字越大，通常效果越好，时间也越久，通常20～100），图片size 调整为 512 x 512 （通常使用512 x 512，或 768 x768，尺寸越大，时间越久）

```
strong warrior princess| centered| key visual| intricate| highly detailed| breathtaking beauty| precise lineart| vibrant| comprehensive cinematic| Carne Griffiths| Conrad Roset
```

如上的prompt词翻译过来是：勇士公主|居中|主视觉|错综复杂|非常详细|美到窒息|精准线稿|充满活力|综合电影|卡恩·格里菲思|康拉德·罗塞特

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image12.png" style="zoom:40%;" />

**现在您可以发挥您的想象力进行创作了！**

为了获取不错的出图效果，创作者们在 prompt词上面会花不少心思，我们可以借鉴别人常用 prompt 词或者复现相关的作品，比如我们常在prompt词中加入如下词组以获得质量高一点的图像：

```
masterpiece, best quality, highly detailed
```

以及常用 Negative prompt 词：

```
lowres, worst quality, ugly, (disfigured), ((mutated hands, misshapen hands, mutated fingers, fused fingers):1.2), extra limbs, deformed legs, disfigured legs, text, logo, watermark

broke a finger, ugly, duplicate, morbid, mutilated, tranny, trans, trannsexual, hermaphrodite, extra fingers, fused fingers, too many fingers, long neck, mutated hands, poorly drawn hands, poorly drawn face, mutation, deformed, blurry, bad anatomy, bad proportions, malformed limbs, extra limbs, cloned face, disfigured, gross proportions, missing arms, missing legs, extra arms, extra legs, artist name, jpeg artifacts
```

#### 3.3 常用网址

AIGC SD 爱好者们非常活跃，模型或者生成的图片层出不穷，推荐常逛如下网站，了解优秀的图像作品用的 prompt 词，随机种子，采样步数等。

[https://lexica.art/](https://lexica.art/)

[https://prompthero.com/](https://lexica.art/)

[https://civitai.com](https://lexica.art/)


### 4 Finetune 风格模型

接下来我们尝试在预训练模型之上进行 finetune（微调），训练一个能出指定风格的模型。

#### 4.1 准备训练素材

我们预先准备了一组日系卡通图片（共20张），下载地址：https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/jp-style.zip 

注意：如果你要使用自己的素材照片，最好统一分辨率为512*512。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image20.png" style="zoom:40%;" />

下载到笔记本电脑后，解压，手动将图片上传到 S3 存储桶（不含 jp-style 文件夹）：

```shell
wget https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/jp-style.zip
unzip jp-style.zip
aws s3 sync ./jp-style/ s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/dataset/
```

#### 4.2 创建训练任务

我们通过 WebUI创建训练任务，首先在**训练**页下面找到训练 **Dreambooth**，然后在***模型***下创建新模型，填入一个名称，**源检查点处**刷新并选择一个模型。然后切换到***参数***。（<u>注意**不要点**最下面的**训练模型**</u>）

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image13.png" style="zoom:40%;" />

在***参数***页下，选择实例类型，建议 **ml.g4dn.2xlarge**, **Concepts S3** 位置填写**4.1**步骤中我们上传到的 S3 路径。然后切换到***概念***页。（<u>注意**不要点**最下面的**训练模型**</u>）

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image14.png" style="zoom:40%;" />

在***概念***页，在**数据集目录处**填写 ```/opt/ml/input/data/concepts/```，在 **Filewords** 处可以分别填写 ```photo of jpstyle girl``` 和 ```photo of girl```。其中 **jpstyle** 可以替换为任意非正常英文单词，可以是字母和数字的组合，如 **asdf1234**。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image15.png" style="zoom:50%;" />

最后在最下面，点击**训练**，看到左侧显示**训练任务提交成功**。

该模型训练预计需要1小时，我们可以在 SageMaker控制台，*Training/Training Jobs* 处看到当前训练任务及其状态。等该 job 标记为 Completed，我们可以开始下一步操作。

与此同时，您可以继续在 WebUI 中探索文生图，或者图生图等。

#### 4.3 部署上面 finetune 的模型

当该训练任务完成，我们可以在 S3 中找到生成的模型文件，路径是：(注意修改下面命令中 S3桶名为自己的桶，替换 account id 即可)

```shell
s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/models/
```

我们取最后一个模型文件（比如jp-style-model_2500），主要 ckpt 和 yaml文件，用于接下来的部署。



<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image16.png" style="zoom:50%;" />



我们回到前面创建的 EC2，通过 awscli 将上面两个文件下载，打包成 model.tar.gz 文件，然后再上传到 S3。

```shell
cd ~ubuntu
mkdir Stable-diffusion
mkdir ControlNet
```

下载模型文件：

```shell
aws s3 cp s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/models/jp-style-model_2500.ckpt ./Stable-diffusion
aws s3 cp s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/models/jp-style-model_2500.yaml ./Stable-diffusion
```

打包：

```shell
tar -czvf model.tar.gz ./Stable-diffusion ./ControlNet
```

最后上传到 SageMaker 默认 S3 存储桶：

```shell
aws s3 cp model.tar.gz s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/assets/jpstyle/model.tar.gz
```

上传完成后，我们再到 All in one ai portal 去部署一个模型端点，操作步骤与 **3.1** 相似，需要注意此时填入最新的 model.tar.gz 的 S3 URI，点击提交，等待约10分钟，待 SageMaker Endpoint 处于 InService 状态即可使用。执行命令获取 S3 URI:

```shell
echo s3://sagemaker-$REGION-$YOUR_ACCOUNT_ID/stable-diffusion-webui/assets/jpstyle/model.tar.gz
```

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image17.png" style="zoom:70%;" />

#### 4.4 在 WebUI 使用 Finetune之后的模型

当 SageMaker Endpoint 处于 InService 状态时，我们切换到 WebUI 界面。在**设置**页面下，找到 **SageMaker Endpoint**，刷新，选中刚刚部署的模型推理端点，在下方刷新，选中模型，最后保存。

![](https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image18.png)

在文生图界面，输入我们 finetune 模型时设定的 prompt 词 ``` jpstyle```，即可生成相同风格的图片。

**注意**：因为我们模型训练用的是 sd 2.1，该模型的出图尺寸建议是 **768x768**，在界面上做相应的调整，如果是 512x512，出图效果可能不佳。

<img src="https://shishuai-share-external.s3.cn-north-1.amazonaws.com.cn/images/aigc-workshop/image19.png" style="zoom:80%;" />


### 5 ControlNet
This feature will be guided by instructors onsite.

### 6 未来尝试

除了今天我们介绍的方式之外，面向算法开发者以及未来大规模生产使用，我们提供了 SageMaker 作为机器学习开发平台，可以通过写代码调用 API 的方式创建模型训练和模型部署以及推理，为了降低使用门槛，我们通过 [SageMaker JumpStart](https://aws.amazon.com/cn/sagemaker/jumpstart/) 提供了快速上手的方式，您可以参考这篇 [blog](https://aws.amazon.com/cn/blogs/machine-learning/fine-tune-text-to-image-stable-diffusion-models-with-amazon-sagemaker-jumpstart/)。

### 7 资源清理

可以先手动删除 SageMaker 推理 Endpoint，然后在 CloudFormation 中删除前面创建的 Stack，相关资源将自动被删除。第一步创建的 EC2 可以 stop掉，或者打个镜像之后 terminate。

### 参考资料

[https://docs.ai.examples.pro/stable-diffusion-webui/](https://docs.ai.examples.pro/stable-diffusion-webui/)

