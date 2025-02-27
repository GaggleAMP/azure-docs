---
title: Migrate from Estimators to ScriptRunConfig
titleSuffix: Azure Machine Learning
description: Migration guide for migrating from Estimators to ScriptRunConfig for configuring training jobs.
services: machine-learning
ms.author: balapv
author: balapv
ms.reviewer: sgilley
ms.service: machine-learning
ms.subservice: core
ms.date: 09/14/2022
ms.topic: how-to
ms.custom: UpdateFrequency5, devx-track-python, contperf-fy21q1, sdkv1, event-tier1-build-2022
---

# Migrating from Estimators to ScriptRunConfig

[!INCLUDE [sdk v1](../../../includes/machine-learning-sdk-v1.md)]

Up until now, there have been multiple methods for configuring a training job in Azure Machine Learning via the SDK, including Estimators, ScriptRunConfig, and the lower-level RunConfiguration.   To address this ambiguity and inconsistency, we are simplifying the job configuration process in Azure Machine Learning.  You should now use ScriptRunConfig as the recommended option for configuring training jobs. 

Estimators are deprecated with the 1.19. release of the Python SDK. You should also generally avoid explicitly instantiating a RunConfiguration object yourself, and instead configure your job using the ScriptRunConfig class.

This article covers common considerations when migrating from Estimators to ScriptRunConfig.

> [!IMPORTANT]
> To migrate to ScriptRunConfig from Estimators, make sure you are using >= 1.15.0 of the Python SDK.

## ScriptRunConfig documentation and samples
Azure Machine Learning documentation and samples have been updated to use [ScriptRunConfig](/python/api/azureml-core/azureml.core.script_run_config.scriptrunconfig) for job configuration and submission.

For information on using ScriptRunConfig, refer to the following documentation:
* [Configure and submit training jobs](how-to-set-up-training-targets.md)
* [Configuring PyTorch training jobs](how-to-train-pytorch.md)
* [Configuring TensorFlow training jobs](how-to-train-tensorflow.md)
* [Configuring scikit-learn training jobs](how-to-train-scikit-learn.md)

In addition, refer to the following samples & tutorials:
* [Azure/MachineLearningNotebooks](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/ml-frameworks)
* [Azure/azureml-examples](https://github.com/Azure/azureml-examples)

## Defining the training environment
While the various framework estimators have preconfigured environments that are backed by Docker images, the Dockerfiles for these images are private.  Therefore you do not have a lot of transparency into what these environments contain. In addition, the estimators take in environment-related configurations as individual parameters (such as `pip_packages`, `custom_docker_image`) on their respective constructors.

When using ScriptRunConfig, all environment-related configurations are encapsulated in the `Environment` object that gets passed into the `environment` parameter of the ScriptRunConfig constructor. To configure a training job,  provide an environment that has all the dependencies required for your training script. If no environment is provided, Azure Machine Learning will use one of the Azure Machine Learning base images, specifically the one defined by `azureml.core.environment.DEFAULT_CPU_IMAGE`, as the default environment. There are a couple of ways to provide an environment:

* [Use a curated environment](../how-to-use-environments.md#use-a-curated-environment) - curated environments are predefined environments available in your workspace by default. There is a corresponding curated environment for each of the preconfigured framework/version Docker images that backed each framework estimator.
* [Define your own custom environment](how-to-use-environments.md)

Here is an example of using the curated environment for training:

```python
from azureml.core import Workspace, ScriptRunConfig, Environment

curated_env_name = '<add Pytorch curated environment name here>'
pytorch_env = Environment.get(workspace=ws, name=curated_env_name)

compute_target = ws.compute_targets['my-cluster']
src = ScriptRunConfig(source_directory='.',
                      script='train.py',
                      compute_target=compute_target,
                      environment=pytorch_env)
```

> [!TIP]
> For a list of curated environments, see [curated environments](../resource-curated-environments.md).  

If you want to specify **environment variables** that will get set on the process where the training script is executed, use the Environment object:
```
myenv.environment_variables = {"MESSAGE":"Hello from Azure Machine Learning"}
```

For information on configuring and managing Azure Machine Learning environments, see:
* [How to use environments](how-to-use-environments.md)
* [Curated environments](../resource-curated-environments.md)
* [Train with a custom Docker image](how-to-train-with-custom-image.md)

## Using data for training
### Datasets
If you are using an Azure Machine Learning dataset for training, pass the dataset as an argument to your script using the `arguments` parameter. By doing so, you will get the data path (mounting point or download path) in your training script via arguments.

The following example configures a training job where the FileDataset, `mnist_ds`, will get mounted on the remote compute.
```python
src = ScriptRunConfig(source_directory='.',
                      script='train.py',
                      arguments=['--data-folder', mnist_ds.as_mount()], # or mnist_ds.as_download() to download
                      compute_target=compute_target,
                      environment=pytorch_env)
```

### DataReference (old)
While we recommend using Azure Machine Learning Datasets over the old DataReference way, if you are still using DataReferences for any reason, you must configure your job as follows:
```python
# if you want to pass a DataReference object, such as the below:
datastore = ws.get_default_datastore()
data_ref = datastore.path('./foo').as_mount()

src = ScriptRunConfig(source_directory='.',
                      script='train.py',
                      arguments=['--data-folder', str(data_ref)], # cast the DataReference object to str
                      compute_target=compute_target,
                      environment=pytorch_env)
src.run_config.data_references = {data_ref.data_reference_name: data_ref.to_config()} # set a dict of the DataReference(s) you want to the `data_references` attribute of the ScriptRunConfig's underlying RunConfiguration object.
```

For more information on using data for training, see:
* [Train with datasets in Azure Machine Learning](how-to-train-with-datasets.md)

## Distributed training
If you need to configure a distributed job for training, do so by specifying the `distributed_job_config` parameter in the ScriptRunConfig constructor. Pass in an [MpiConfiguration](/python/api/azureml-core/azureml.core.runconfig.mpiconfiguration), [PyTorchConfiguration](/python/api/azureml-core/azureml.core.runconfig.pytorchconfiguration), or [TensorflowConfiguration](/python/api/azureml-core/azureml.core.runconfig.tensorflowconfiguration) for distributed jobs of the respective types.

The following example configures a PyTorch training job to use distributed training with MPI/Horovod:
```python
from azureml.core.runconfig import MpiConfiguration

src = ScriptRunConfig(source_directory='.',
                      script='train.py',
                      compute_target=compute_target,
                      environment=pytorch_env,
                      distributed_job_config=MpiConfiguration(node_count=2, process_count_per_node=2))
```

For more information, see:
* [Distributed training with PyTorch](how-to-train-pytorch.md#distributed-training)
* [Distributed training with TensorFlow](how-to-train-tensorflow.md#distributed-training)

## Miscellaneous
If you need to access the underlying RunConfiguration object for a ScriptRunConfig for any reason, you can do so as follows:
```
src.run_config
```

## Next steps

* [Configure and submit training jobs](how-to-set-up-training-targets.md)
