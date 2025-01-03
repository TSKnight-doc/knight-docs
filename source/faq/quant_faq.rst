=====================
量化工具FAQ
=====================

本文档主要解答量化过程中常见问题及注意事项，为产品应用提供技术支持。

量化工具精度调优策略
======================

量化校准数据集选取
-------------------

模型量化是一个概率模型，通过校准数据集，统计量化参数，用低比特来近似表达原模型，所以校准数据集的选取本身对模型的量化是非常重要的，往往遇到一些量化精度问题时，需要尝试用不同的校准数据集来调优量化精度。

量化数据集的选取没有统一的标准，但我们也发现一些规律，比如分类任务在Imagenet数据集上，一般随机采样500-1000张图片可以达到一个较高的量化精度。在另外一些语音的任务中，我们也发现，采用更少量的数据也会达到一个更好的量化效果，比如100条语音数据在编解码任务中。

综上，数据量的选取很大程度是量化的一个超参数，需要根据实际任务进行调优。除了校准集的数量，数据质量也很重要。选校准集时，要尽可能包含各个场景的数据，比如分类任务尽量每个类别的数据都包含一些。

从量化的角度来讲，通过校准集统计出来的模型，能否最大化的表示浮点模型，决定着最终的量化效果。

量化系数统计方式
----------------

Onnx量化工具支持KL散度，min_max，mse，
pertencile和kl_v2多种统计量化系数的方式。根据量化误差理论，一般而言，8bit量化选用KL散度量化效果好，16bit采用min_max方式量化效果好。但这也不是绝对的，还取决于模型每层激活值的分布类型。所以实际操作时，建议分别从所有支持的系数统计方式进行量化，选择更优者。

小技巧一: 在校准集的数据量选择上，min_max往往比KL散度需要更少的数据。

小技巧二: 8bit量化模式下，可以优先尝试使用kl_v2, kl和mse进行量化。

模型输入是整型值时的处理方法
----------------------------

当模型的输入数据本身就是整数，不对输入数据进行量化，效果会更好。下面详细说明：

1）输入数据是0-255或者-128-127之间的整数时，8bit/16bit都是建议不对输入进行量化的。

2）输入数据是-32768-32767之间的整数时，16bit是建议不对输入进行量化的，8bit还是需要进行量化。

量化模型输出接后处理的方法
--------------------------

因为量化模型的输出是8bit/16bit的整型数值，而后处理一般都是浮点运算，所以需要把量化模型的整型数值乘以一个量化系数还原成浮点，这点需要注意。不同的量化框架操作有所不同，具体操作可参考用户指南相关章节。

ONNX量化工具FAQ
===============

量化典型报错汇总
----------------

输入数据与模型输入定义的shape不一致引发的错误
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+--------------+-------------------------------------------------------+
| 问题         | [ONNXRuntimeError] : 2 : INVALID_ARGUMENT : Got       |
|              | invalid dimensions for input: data_input for the      |
|              | following indices                                     |
|              |                                                       |
|              | index: 0 Got: 128 Expected: 1                         |
|              |                                                       |
|              | Please fix either the inputs or the model.            |
+==============+=======================================================+
| 出现原因     | 输入和定义的shape大小不一致导致前向推理报错，         |
|              | 模型的输入shape第一维定义为1而输入的数据第一维为128。 |
+--------------+-------------------------------------------------------+
| 解决方案     | 加载数据时与模型定义中对应的输入的s                   |
|              | hape保持一致即可，例如上述问题将输入数据第一维改为1。 |
+--------------+-------------------------------------------------------+

量化过程中计算出的q值未在硬件限制规格内引发的异常退出
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+--------------+-------------------------------------------------------+
| 问题         | AssertionError OP:Conv_scaleFix q2:-12 must range in  |
|              | [0, 31] to meet handware specifications               |
+==============+=======================================================+
| 出现原因     | 上述错误中出现的q值为后续硬件计                       |
|              | 算所使用，错误信息表示当前量化过程中计算出的q值已超出 |
|              | 硬件规格限制[0,31]，量化后模型无法在硬件芯片上运行，  |
|              | 量化阶段无法解决此类异常，量化过程提前终止程序退出。  |
+--------------+-------------------------------------------------------+
| 解决方案     | 1：更换量化数据集重新开展量化。                       |
|              | 2：使用Fi                                             |
|              | netune库进行qat模型训练，调整模型权重参数后再次量化。 |
+--------------+-------------------------------------------------------+

量化过程相关问题答疑
--------------------

量化过程中打印出的Quantization Loss的计算方式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    Loss = abs(fix_acurracy - float_acurracy)/abs(float_acurracy) \* 100

其中float_acurracy和fix_acurracy分别指浮点模型和定点模型执行前向推理时由用户自定义的前向推理函数返回的结果值。

量化过程中出现lose precison的Warning信息
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

例如日志中出现以下Warning信息：

OP:node_GRU_0_xn, q_rst: -1 < 0, fix it use q_rst=0, this will lose
precision

出现此种信息是因为当前模型参数和量化数据集得到的定点参数值小于0，底层硬件规格不支持调整为0可保证硬件可支持，但此调整可能带来精度的下降，具体请验证量化后模型的精度，如不满足要求则需调整量化数据集或者使用Finetune工具或者重训练调整模型参数再次开展量化。

量化数据的加载顺序对量化结果是否有影响
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

因为Conv等算子在定点实现过程中存在中间和与相应q值的计算，数据的加载顺序可能导致中间和对应的q值不一致进而影响定点结果不相同，但并非数据加载顺序不一致会一定导致最终量化结果不相同，只是定点实现的方案会有几率出现最终量化后模型的定点q值不一致进而量化结果存在一定差异。

非npy格式数据的infer函数实现
----------------------------

使用ONNX量化工具进行模型量化时需要用户自行实现infer函数的编写，使用指南中的样例是以预处理后保存的npy格式数据为例，若为其他格式的数据只需要参照模型训练流程中的预处理流程将数据处理成numpy格式数据，然后执行模型前向预测executor.forward（data），基于输出结果计算相对应的精度指标即可，ONNX量化工具会根据量化前后的精度指标自动计算量化精度下降loss，若量化过程中不关注精度下降指标亦可省略计算相对应的精度指标即后处理模型部分，直接在infer函数返回固定数值例如1即可。

若量化数据为图片数据，待量化模型为CNN分类网络时，infer函数实现如下(imagenet_benchmark)：

.. code-block:: python

    import os
    import time
    import torch
    import torchvision.transforms as transforms
    import torchvision.datasets as datasets
    import numpy as np

    from timm.utils import accuracy, AverageMeter
    from torchvision import datasets, transforms


    class AverageMeter(object):
        """Computes and stores the average and current value"""

        def __init__(self):
            self.reset()

        def reset(self):
            self.val = 0
            self.avg = 0
            self.sum = 0
            self.count = 0

        def update(self, val, n=1):
            self.val = val
            self.sum += val * n
            self.count += n
            self.avg = self.sum / self.count


    def accuracy(output, target, topk=(1,)):
        """Computes the precision@k for the specified values of k"""
        maxk = max(topk)
        batch_size = target.size(0)
        _, pred = output.topk(maxk, 1, largest=True, sorted=True)
        pred = pred.t()
        correct = pred.eq(target.reshape(1, -1).expand_as(pred))

        res = []
        for k in topk:
            correct_k = correct[:k].reshape(-1).float().sum(0, keepdim=True)
            res.append(correct_k.mul_(100.0 / batch_size))
        return res

    def imagenet_benchmark(executor, crop_size=224):
        iters = executor.iteration
        batch_size = executor.batch_size
        valdir = '/data/public_data/imagenet/images/val'
        normalize = transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])

        val_dataset = datasets.ImageFolder(
            valdir,
            transforms.Compose([
                transforms.Resize(256),
                transforms.CenterCrop(crop_size),
                transforms.ToTensor(),
                normalize,
            ]))
        val_loader = torch.utils.data.DataLoader(
            val_dataset, batch_size=batch_size, shuffle=False,
            num_workers=4, pin_memory=True, sampler=None)

        batch_time = AverageMeter()
        top1 = AverageMeter()
    top5 = AverageMeter()
    end = time.time()

        for i, (input, label) in enumerate(val_loader):
            input_numpy = input.numpy()
            output = executor.forward(input_numpy)
            output = torch.from_numpy(output[0]).data

            # measure accuracy and record loss
            prec1, prec5 = accuracy(output, label, topk=(1, 5))
            top1.update(prec1[0], input.size(0))
            top5.update(prec5[0], input.size(0))

            # measure elapsed time
            batch_time.update(time.time() - end)

            print('Test: [{0}/{1}]\t'
                'Time {batch_time.val:.3f} ({batch_time.avg:.3f})\t'
                'Prec@1 {top1.val:.3f} ({top1.avg:.3f})\t'
                'Prec@5 {top5.val:.3f} ({top5.avg:.3f})'.format(
                i, iters, batch_time=batch_time,
                top1=top1, top5=top5))

            if i+1 == iters:
                break
        print(' * Prec@1 {top1.avg:.3f} Prec@5 {top5.avg:.3f}'.format(top1=top1, top5=top5))
        return top1.avg.item()


自动根据输入shape生成随机数据开展量化
-------------------------------------

目前量化工具内部已内置了此自动根据输入shape生成随机数据开展量化的infer函数(具体可查看/opt
/Quantize/Onnx_examples/infer_auto.py)，若不关注模型量化精度只关注整体量化流程，可在量化时指定-if
infer_auto自动开展量化。

.. code-block:: python

    import numpy as np
    from onnx_quantize_tool.common.register import onnx_infer_func

    type_dict = {
        1: np.float32,
        2: np.uint8,
        3: np.int8,
        4: np.uint16,
        5: np.int16,
        6: np.int32,
        7: np.int64,
        10: np.float16,
        11: np.float64,
        12: np.uint32,
        13: np.uint64
    }

    def _get_random_by_shape_type(shape, dtype):
        np.random.seed(123)
        # float
        if dtype in [1, 10, 11]:
            data = np.random.randn(*shape)
        # int
        elif dtype in [3, 5, 6, 7]:
            data = np.random.randint(-10, 10, size=shape)
        # uint
        elif dtype in [2, 4, 12, 13]:
            data = np.random.randint(1, 10, size=shape)
        else:
            raise RuntimeError(f'Unsupport dtype:{dtype}')

        return data

    @ onnx_infer_func.register("infer_auto")
    def infer_auto(executor):
        iteration = executor.iteration
        batch_size = executor.batch_size
        input_names = executor.input_nodes
        # dict{key:input-name, value:input-type}
        input_dtypes = executor.input_types
        if not executor.shape_dicts:
            executor.init_shape_info()
        # dict{key:input-name, value:如[1,3,224,224]}
        input_shapes = executor.shape_dicts
        datas = []
        for input_name in input_names:
            ori_shape = input_shapes[input_name]
            # 动态shape自动转为batch_size
            input_shape = [shape if isinstance(shape, int) and shape > 0 else batch_size for shape in ori_shape]
            # 默认是float32
            input_dtype = input_dtypes.get(input_name, 1)
            data = _get_random_by_shape_type(input_shape, input_dtype)
            datas.append(data.astype(type_dict[input_dtype]))

        for _ in range(iteration):
            executor.forward(*datas)
        return None




获取量化后模型的所有中间层结果
------------------------------

量化完成后，会生成能够获取所有中间层结果的模型，其存放于{-s指定的存放目录}/steps/
``output_all_layers_quantize.onnx`` ，只需加载此模型执行前向推理即可获取所有中间层结果，即 ``infer`` 函数中的 ``output=executor.forward(input_data)`` 可获取到所有中间层结果，结果对应的输出层名可通过 ``executor.output_node_list`` 获取，此两个 ``List`` 数目相同且一一对应。

.. note::

    需先进行量化完成之后才会产生此模型

.. code-block:: bash

    Knight --chip TX5105C quant onnx -s tmp/resnet18 
    -m /tmp/resnet18/resnet18/steps/output_all_layers_quantize.onnx 
    -if infer_auto -qm kl -bs 10 -r infer -i 1


