# PyTorch ML Lifecycle

Learn about the ML model lifecycle with PyTorch, how to API-ify those models, and deploy them to EKS with Astrobase.

This example is more involved, and takes about an hour to work through.

For a faster walkthrough on how to deploy pre-made models, visit [this example](../pytorch-model-deployment).

Here's a table of contents incase you decide to put a bookmark in your progress.

MLP = Multilayer Perceptrons
CNN = Convolutional Neural Networks

1. [Installing PyTorch](#installing-pytorch)
1. [PyTorch Deep-Learning Model Life-Cycle](#pytorch-deep-learning-model-life-cycle)
1. [Develop PyTorch Deep Learning Models](#develop-pytorch-deep-learning-models)
    1. [Develop an MLP for Binary Classification](#develop-an-mlp-for-binary-classification)
    1. [Develop an MLP for Multiclass Classification](#develop-an-mlp-for-multiclass-classification)
    1. [Develop an MLP for Regression](#develop-an-mlp-for-regression)
    1. [Develop a CNN for Image Classification](#develop-a-cnn-for-image-classification)
1. [Deploying PyTorch Models to EKS with Astrobase](#deploying-pytorch-models-to-eks-with-astrobase)

We'll be using python 3.9.0 for this example.

## Installing PyTorch

Create a virtual environment and install PyTorch and a few libraries we'll need for running our models and formatting code.

```sh
$ python -m venv .venv
$ source .venv/bin/activate
$ python -m pip install torch torchvision numpy pandas sklearn black flake8 mypy
```

We won't be using GPUs for this exercise, but you can provision GPU nodegroups and nodepools on EKS and GKE with Astrobase.

Confirm that you've installed pytorch.

```sh
$ python -c "import torch; print(torch.__version__)"
1.8.1
```

## PyTorch Deep-Learning Model Life-Cycle

Here are five steps of the ML Model Life-Cycle we will cover:

1. [Prepare the Data](#prepare-the-data)
1. [Define the Model](#define-the-model)
1. [Train the Model](#trainp-the-model)
1. [Evaluate the Model](#evaluate-the-model)
1. [Make Predictions](#make-predictions)

### Prepare the Data

You can use standard Python libraries to load and prepare tabular data, like CSV files. For example, Pandas can be used to load your CSV file, and tools from scikit-learn can be used to encode categorical data, such as class labels.

PyTorch provides the [Dataset](https://pytorch.org/docs/stable/data.html#torch.utils.data.Dataset) class that you can extend and customize to load your dataset.

For example, the constructor of your dataset object can load your data file (e.g. a CSV file). You can then override the `__len__()` function that can be used to get the length of the dataset (number of rows or samples), and the `__getitem__()` function that is used to get a specific sample by index.

When loading your dataset, you can also perform any required transforms, such as scaling or encoding.

A skeleton of a custom Dataset class is provided below.

P.S. – Don't worry about writing all of this out now, we'll do that with complete examples below. If you'd like to skip ahead to those examples, please jump [here](#develop-pytorch-deep-learning-models).

```py
class CSVDataset(Dataset):
    # initialize and load the dataset
    def __init__(self, path: str, ...):
        # store the inputs and outputs
        self.x = ...
        self.y = ...

    # number of rows in the dataset
    def __len__(self):
        return len(self.x)

    # get a row at an index
    def __getitem__(self, idx):
        return [self.x[idx], self.y[idx]]
```

Once loaded, PyTorch provides the [DataLoader](https://pytorch.org/docs/stable/data.html#torch.utils.data.DataLoader) class to navigate a Dataset instance during the training and evaluation of your model.

A DataLoader instance can be created for the training dataset, test dataset, and even a validation dataset.

The random_split() function can be used to split a dataset into train and test sets. Once split, a selection of rows from the Dataset can be provided to a DataLoader, along with the batch size and whether the data should be shuffled every epoch.

For example, we can define a DataLoader by passing in a selected sample of rows in the dataset.

```py
# create the dataset
dataset = CSVDataset(...)
# select rows from the dataset
train, test = random_split(dataset, [[...], [...]])
# create a data loader for train and test sets
train_dl = DataLoader(train, batch_size=32, shuffle=True)
test_dl = DataLoader(test, batch_size=1024, shuffle=False)
```

Once defined, a DataLoader can be enumerated, yielding one batch worth of samples each iteration.

```py
...
# train the model
for i, (inputs, targets) in enumerate(train_dl):
	...
```

### Define the Model

The idiom for defining a model in PyTorch involves defining a class that extends the [Module](https://pytorch.org/docs/stable/nn.html#module) class.

The constructor of your class defines the layers of the model and the forward() function is the override that defines how to forward propagate input through the defined layers of the model.

Many layers are available, such as [Linear](https://pytorch.org/docs/stable/nn.html#torch.nn.Linear) for fully connected layers, [Conv2d](https://pytorch.org/docs/stable/nn.html#torch.nn.Conv2d) for convolutional layers, and [MaxPool2d](https://pytorch.org/docs/stable/nn.html#torch.nn.MaxPool2d) for pooling layers.

Activation functions can also be defined as layers, such as [ReLU](https://pytorch.org/docs/stable/nn.html#torch.nn.ReLU), [Softmax](https://pytorch.org/docs/stable/nn.html#torch.nn.Softmax), and [Sigmoid](https://pytorch.org/docs/stable/nn.html#torch.nn.Sigmoid).

Here is an example of a simple MLP model with one layer.

```py
class MLP(Module):
    # define model elements
    def __init__(self, n_inputs):
        super(MLP, self).__init__()
        self.layer = Linear(n_inputs, 1)
        self.activation = Sigmoid()

    # forward propagate input
    def forward(self, x):
        x = self.layer(x)
        x = self.activation(x)
        return x
```

### Train the Model

The training process requires that you define a loss function and an optimization algorithm.

Common loss functions include the following:

[BCELoss](https://pytorch.org/docs/stable/nn.html#torch.nn.BCELoss): Binary cross-entropy loss for binary classification.
[CrossEntropyLoss](https://pytorch.org/docs/stable/nn.html#torch.nn.CrossEntropyLoss): Categorical cross-entropy loss for multi-class classification.
[MSELoss](https://pytorch.org/docs/stable/nn.html#torch.nn.MSELoss): Mean squared loss for regression.

If you're searching for what kind of loss and optimization functions to use, it's best to consult pytorch's documentation, there really is plenty of resources to help you identify the right functions and algorithms for your models/products needs.

Stochastic gradient descent is used for optimization, and the standard algorithm is provided by the [SGD](https://pytorch.org/docs/stable/optim.html#torch.optim.SGD) class, although other versions of the algorithm are available, such as [Adam](https://pytorch.org/docs/stable/optim.html#torch.optim.Adam).

```py
# define the optimization
criterion = MSELoss()
optimizer = SGD(model.parameters(), lr=0.01, momentum=0.9)
```

Training the model involves enumerating the DataLoader for the training dataset.

First, a loop is required for the number of training epochs. Then an inner loop is required for the mini-batches for stochastic gradient descent.

```py
...
# enumerate epochs
for epoch in range(100):
    # enumerate mini batches
    for i, (inputs, targets) in enumerate(train_dl):
    	...
```

Each update to the model involves the same general pattern comprised of:

1. Clearing the last error gradient.
1. A forward pass of the input through the model.
1. Calculating the loss for the model output.
1. Backpropagating the error through the model.
1. Update the model in an effort to reduce loss.

For example:

```py
# clear the gradients
optimizer.zero_grad()
# compute the model output
yhat = model(inputs)
# calculate loss
loss = criterion(yhat, targets)
# credit assignment
loss.backward()
# update model weights
optimizer.step()
```


### Evaluate the Model

Once the model is fit, it can be evaluated on the test dataset.

This can be achieved by using the DataLoader for the test dataset and collecting the predictions for the test set, then comparing the predictions to the expected values of the test set and calculating a performance metric.

```py
...
for i, (inputs, targets) in enumerate(test_dl):
    # evaluate the model on the test set
    yhat = model(inputs)
    ...
```

### Make Predictions

A fit model can be used to make a prediction on new data.

For example, you might have a single image or a single row of data and want to make a prediction.

This requires that you wrap the data in a PyTorch Tensor data structure.

A Tensor is just the PyTorch version of a NumPy array for holding data. It also allows you to perform the automatic differentiation tasks in the model graph, like calling backward() when training the model.

The prediction too will be a Tensor, although you can retrieve the NumPy array by detaching the Tensor from the automatic differentiation graph and calling the NumPy function.

```py
...
# convert row to data
row = Variable(Tensor([row]).float())
# make prediction
yhat = model(row)
# retrieve numpy array
yhat = yhat.detach().numpy()
```

Now that we are familiar with the PyTorch API at a high-level and the model life-cycle, let’s look at how we can develop some standard deep learning models from scratch!

## Develop PyTorch Deep Learning Models

In this section, you will discover how to develop, evaluate, and make predictions with standard deep learning models, including Multilayer Perceptrons (MLP) and Convolutional Neural Networks (CNN).

A Multilayer Perceptron model, or MLP for short, is a standard fully connected neural network model.

It is comprised of layers of nodes where each node is connected to all outputs from the previous layer and the output of each node is connected to all inputs for nodes in the next layer.

An MLP is a model with one or more fully connected layers. This model is appropriate for **tabular data**, that is data as it looks in a table or spreadsheet with one column for each variable and one row for each variable. There are three predictive modeling problems you may want to explore with an MLP; they are binary classification, multiclass classification, and regression.

Let’s fit a model on a real dataset for each of these cases.

### Develop an MLP (Multilayer Perceptrons) for Binary Classification

We will use the Ionosphere binary (two class) classification dataset to demonstrate an MLP for binary classification.

This dataset involves predicting whether there is a structure in the atmosphere or not given radar returns.

The dataset will be downloaded automatically using Pandas.

We will use a [LabelEncoder](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.LabelEncoder.html) to encode the string labels to integer values 0 and 1. The model will be fit on 67 percent of the data, and the remaining 33 percent will be used for evaluation, split using the [train_test_split()](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html) function.

It is a good practice to use ‘relu‘ activation with a ‘He Uniform‘ weight initialization. This combination goes a long way to overcome the problem of [vanishing gradients](https://machinelearningmastery.com/how-to-fix-vanishing-gradients-using-the-rectified-linear-activation-function/) when training deep neural network models.

A Gentle Introduction to the Rectified Linear Unit (ReLU)
The model predicts the probability of class 1 and uses the sigmoid activation function. The model is optimized using stochastic gradient descent and seeks to minimize the [binary cross-entropy loss](https://machinelearningmastery.com/cross-entropy-for-machine-learning/).

The complete example is listed [here](./mlp_binary_ionosphere.py).

To run the example,

```sh
$ python mlp_binary_ionosphere.py
```

### Develop an MLP for Multiclass Classification

We will use the Iris flowers multiclass classification dataset to demonstrate an MLP for multiclass classification.

This problem involves predicting the species of iris flower given measures of the flower.

The dataset will be downloaded automatically using Pandas.

Given that it is a multiclass classification, the model must have one node for each class in the output layer and use the softmax activation function. The loss function is the cross entropy, which is appropriate for integer encoded class labels (e.g. 0 for one class, 1 for the next class, etc.).

The complete example of fitting and evaluating an MLP on the iris flowers dataset is listed [here](./mlp_multiclass_iris.py).

To run the example,

```sh
$ python mlp_multiclass_iris.py
```


### Develop an MLP for Regression

We will use the Boston housing regression dataset to demonstrate an MLP for regression predictive modeling.

This problem involves predicting house value based on properties of the house and neighborhood.

The dataset will be downloaded automatically using Pandas.

This is a regression problem that involves predicting a single numeric value. As such, the output layer has a single node and uses the default or linear activation function (no activation function). The mean squared error (mse) loss is minimized when fitting the model.

Recall that this is regression, not classification; therefore, we cannot calculate classification accuracy.

The complete example of fitting and evaluating an MLP on the Boston housing dataset is [here](./mlp_regression_housing.py).

To run the example,

```sh
$ python mlp_regression_housing.py
```

### Develop a CNN (Convolutional Neural Networks) for Image Classification

Convolutional Neural Networks, or CNNs for short, are a type of network designed for image input.

They are comprised of models with convolutional layers that extract features (called feature maps) and pooling layers that distill features down to the most noticeable elements.

CNNs are best suited to image classification tasks, although they can be used on a wide array of tasks that take images as input.

A popular image classification task is the [MNIST handwritten digit classification](https://en.wikipedia.org/wiki/MNIST_database). It involves tens of thousands of handwritten digits that must be classified as a number between 0 and 9.

The torchvision API provides a convenience function to download and load this dataset directly.

This example loads the dataset and plots the first few images.

```py
# load mnist dataset in pytorch
from torch.utils.data import DataLoader
from torchvision.datasets import MNIST
from torchvision.transforms import Compose
from torchvision.transforms import ToTensor
from matplotlib import pyplot
# define location to save or load the dataset
path = '~/.torch/datasets/mnist'
# define the transforms to apply to the data
trans = Compose([ToTensor()])
# download and define the datasets
train = MNIST(path, train=True, download=True, transform=trans)
test = MNIST(path, train=False, download=True, transform=trans)
# define how to enumerate the datasets
train_dl = DataLoader(train, batch_size=32, shuffle=True)
test_dl = DataLoader(test, batch_size=32, shuffle=True)
# get one batch of images
i, (inputs, targets) = next(enumerate(train_dl))
# plot some images
for i in range(25):
	# define subplot
	pyplot.subplot(5, 5, i+1)
	# plot raw pixel data
	pyplot.imshow(inputs[i][0], cmap='gray')
# show the figure
pyplot.show()
```

We can train a CNN model to classify the images in the MNIST dataset.

Note that the images are arrays of grayscale pixel data, therefore, we must add a channel dimension to the data before we can use the images as input to the model.

It is a good idea to scale the pixel values from the default range of 0-255 to have a zero mean and a standard deviation of 1

The complete example of fitting and evaluating a CNN model on the MNIST dataset is [here](./cnn_mnist.py).

To run the example,

```sh
$ python cnn_mnist.py
```

This may take a few minutes depending on your computer's CPU. It took a test 2013 Macbook Pro 2.3 GHz Quad-Core Intel Core i7 about 4 minutes (it's an old Macbook 🦖).

## Deploying PyTorch Models to EKS with Astrobase

So now all of this is fine and dandy ... but what if we want to deploy these features to AWS?

Now, we have to gather these scripts into a containerized API and deploy them to EKS with Astrobase.

Note that you might not want to use these examples as-is for production. Why? Well, you should be referencing external data stores and use multiple environments. What's great about Astrobase is that it's easy to build and shutdown training environments, so you would need to do some extra work to parameterize the dataset references and training steps. If you and your team need help doing this, [get in touch with us](mailto:anthony@astrobase.co)!

__TODO__

**Credits**
- https://machinelearningmastery.com/pytorch-tutorial-develop-deep-learning-models/
